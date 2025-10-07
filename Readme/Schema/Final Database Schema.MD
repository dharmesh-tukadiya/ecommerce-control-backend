Final Database Schema
1. Shipments Table (fks_shipments)
CREATE TABLE fks_shipments (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shippingId VARCHAR(50) NOT NULL UNIQUE,
    dispatchByDate DATETIME,
    dispatchByDateWithGrace DATETIME,
    dispatchAfterDate DATETIME,
    dispatchServiceTier VARCHAR(50),
    packagingPolicy VARCHAR(50),
    isLabelPrinted BOOLEAN,
    isLarge BOOLEAN,
    isMps BOOLEAN,
    on_hold BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

2. Shipment Order Items Table (fks_shipment_order_items)
CREATE TABLE fks_shipment_order_items (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shipment_id BIGINT UNSIGNED NOT NULL,
    orderItemId VARCHAR(50) NOT NULL,
    enforcedGroupId VARCHAR(50),
    status VARCHAR(50),
    quantity INT,
    totalPrice DECIMAL(12,2),
    imeiCount INT,
    FOREIGN KEY (shipment_id) REFERENCES fks_shipments(id),
    UNIQUE KEY uniq_orderitem_shipment (shipment_id, orderItemId)
);

3. Order Item Details Table (fks_order_item_details)
CREATE TABLE fks_order_item_details (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shipment_order_item_id BIGINT UNSIGNED NOT NULL,
    orderItemId VARCHAR(50) NOT NULL,
    orderId VARCHAR(50),
    quantity INT,
    status VARCHAR(50),
    FOREIGN KEY (shipment_order_item_id) REFERENCES fks_shipment_order_items(id),
    UNIQUE KEY uniq_order_item_detail (shipment_order_item_id, orderItemId)
);

4. Listing Table (fks_listing)
CREATE TABLE fks_listing (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_item_detail_id BIGINT UNSIGNED NOT NULL,
    listingId VARCHAR(50) NOT NULL,
    FOREIGN KEY (order_item_detail_id) REFERENCES fks_order_item_details(id),
    UNIQUE KEY uniq_listing_order_item (order_item_detail_id, listingId)
);

5. Product Table (fks_products)
CREATE TABLE fks_products (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    listing_id BIGINT UNSIGNED NOT NULL,
    productId VARCHAR(50),
    title TEXT,
    productUrl TEXT,
    primaryImageUrl TEXT,
    fsn VARCHAR(50),
    sku VARCHAR(50),
    hsn VARCHAR(50),
    vertical VARCHAR(50),
    FOREIGN KEY (listing_id) REFERENCES fks_listing(id)
);

6. Buyer Details Table (fks_buyer_details)
CREATE TABLE fks_buyer_details (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_item_detail_id BIGINT UNSIGNED NOT NULL,
    name VARCHAR(255),
    phone VARCHAR(50),
    FOREIGN KEY (order_item_detail_id) REFERENCES fks_order_item_details(id)
);

7. Buyer Address Table (fks_buyer_addresses)
CREATE TABLE fks_buyer_addresses (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    buyer_detail_id BIGINT UNSIGNED NOT NULL,
    line1 TEXT,
    line2 TEXT,
    city VARCHAR(100),
    pincode VARCHAR(20),
    state VARCHAR(50),
    FOREIGN KEY (buyer_detail_id) REFERENCES fks_buyer_details(id)
);

8. Offers Table (fks_offers)
CREATE TABLE fks_offers (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shipment_order_item_id BIGINT UNSIGNED NOT NULL,
    offerId VARCHAR(50),
    adjustmentType VARCHAR(50),
    isSellerPromotion BOOLEAN,
    offerDescription TEXT,
    offerValueInRupees DECIMAL(12,2),
    promotionId VARCHAR(50),
    sellerContribution DECIMAL(12,2),
    FOREIGN KEY (shipment_order_item_id) REFERENCES fks_shipment_order_items(id),
    UNIQUE KEY uniq_offer_orderitem (shipment_order_item_id, offerId)
);

9. Tracking Table (fks_tracking)
CREATE TABLE fks_tracking (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shipment_id BIGINT UNSIGNED NOT NULL,
    trackingId VARCHAR(50) NOT NULL,
    courierName VARCHAR(100),
    size VARCHAR(50),
    FOREIGN KEY (shipment_id) REFERENCES fks_shipments(id),
    UNIQUE KEY uniq_tracking_shipment (shipment_id, trackingId)
);

10. Sub Shipments Table (fks_sub_shipments)
CREATE TABLE fks_sub_shipments (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shipment_id BIGINT UNSIGNED NOT NULL,
    externalSubShipmentId VARCHAR(50),
    FOREIGN KEY (shipment_id) REFERENCES fks_shipments(id),
    UNIQUE KEY uniq_subshipment_shipment (shipment_id, externalSubShipmentId)
);

11. Packages Table (fks_packages)
CREATE TABLE fks_packages (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sub_shipment_id BIGINT UNSIGNED NOT NULL,
    packageId VARCHAR(50),
    length DECIMAL(10,2),
    breadth DECIMAL(10,2),
    height DECIMAL(10,2),
    weight DECIMAL(10,2),
    FOREIGN KEY (sub_shipment_id) REFERENCES fks_sub_shipments(id),
    UNIQUE KEY uniq_package_subshipment (sub_shipment_id, packageId)
);

12. IMEI Table (fks_imei_details)
CREATE TABLE fks_imei_details (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shipment_order_item_id BIGINT UNSIGNED NOT NULL,
    imei VARCHAR(50),
    FOREIGN KEY (shipment_order_item_id) REFERENCES fks_shipment_order_items(id)
);


<?php

/**
 * Transform Flipkart shipment JSON into table => data arrays.
 *
 * Example:
 * [
 *   "fks_shipments" => [...],
 *   "fks_shipment_order_items" => [...],
 *   "fks_offers" => [...]
 * ]
 */

function parseFlipkartShipments(array $shipments): array
{
    $result = [
        "fks_shipments"            => [],
        "fks_shipment_order_items" => [],
        "fks_order_item_details"   => [],
        "fks_listing"              => [],
        "fks_products"             => [],
        "fks_buyer_details"        => [],
        "fks_buyer_addresses"      => [],
        "fks_offers"               => [],
        "fks_tracking"             => [],
        "fks_sub_shipments"        => [],
        "fks_packages"             => [],
        "fks_imei_details"         => [],
    ];

    foreach ($shipments as $shipment) {
        // --- fks_shipments
        $shipmentId = $shipment['shippingId'];
        $result["fks_shipments"][] = [
            "shippingId"              => $shipment['shippingId'],
            "dispatchByDate"          => $shipment['dispatchByDate'] ?? null,
            "dispatchByDateWithGrace" => $shipment['dispatchByDateWithGrace'] ?? null,
            "dispatchAfterDate"       => $shipment['dispatchAfterDate'] ?? null,
            "dispatchServiceTier"     => $shipment['dispatchServiceTier'] ?? null,
            "packagingPolicy"         => $shipment['packagingPolicy'] ?? null,
            "isLabelPrinted"          => $shipment['isLabelPrinted'] ?? null,
            "isLarge"                 => $shipment['isLarge'] ?? null,
            "isMps"                   => $shipment['isMps'] ?? null,
            "on_hold"                 => $shipment['on_hold'] ?? null,
        ];

        // --- fks_tracking
        if (!empty($shipment['tracking'])) {
            $tracking = $shipment['tracking'];
            $result["fks_tracking"][] = [
                "shipment_id" => $shipmentId,
                "trackingId"  => $tracking['trackingId'],
                "courierName" => $tracking['courierName'] ?? null,
                "size"        => $tracking['size'] ?? null,
            ];
        }

        // --- fks_shipment_order_items
        foreach ($shipment['shipmentOrderItems'] as $item) {
            $orderItemId = $item['orderItemId'];
            $result["fks_shipment_order_items"][] = [
                "shipment_id"     => $shipmentId,
                "orderItemId"     => $item['orderItemId'],
                "enforcedGroupId" => $item['enforcedGroupId'] ?? null,
                "status"          => $item['status'] ?? null,
                "quantity"        => $item['quantity'] ?? null,
                "totalPrice"      => $item['pricing']['totalPrice'] ?? null,
                "imeiCount"       => $item['imeiDetails']['imeiCount'] ?? 0,
            ];

            // --- fks_order_item_details
            $detail = $item['orderItem'];
            $result["fks_order_item_details"][] = [
                "shipment_order_item_id" => $orderItemId,
                "orderItemId"            => $detail['orderItemId'],
                "orderId"                => $detail['orderId'],
                "quantity"               => $detail['quantity'] ?? null,
                "status"                 => $detail['status'] ?? null,
            ];

            // --- fks_listing
            if (!empty($detail['listing'])) {
                $listingId = $detail['listing']['listingId'];
                $result["fks_listing"][] = [
                    "order_item_detail_id" => $orderItemId,
                    "listingId"            => $listingId,
                ];

                // --- fks_products
                if (!empty($detail['listing']['product'])) {
                    $product = $detail['listing']['product'];
                    $result["fks_products"][] = [
                        "listing_id"      => $listingId,
                        "productId"       => $product['productId'] ?? null,
                        "title"           => $product['title'] ?? null,
                        "productUrl"      => $product['productUrl'] ?? null,
                        "primaryImageUrl" => $product['primaryImageUrl'] ?? null,
                        "fsn"             => $product['fsn'] ?? null,
                        "sku"             => $product['sku'] ?? null,
                        "hsn"             => $product['hsn'] ?? null,
                        "vertical"        => $product['vertical'] ?? null,
                    ];
                }
            }

            // --- fks_buyer_details
            if (!empty($detail['buyerDetails'])) {
                $buyer = $detail['buyerDetails'];
                $buyerId = md5($orderItemId . ($buyer['phone'] ?? ''));
                $result["fks_buyer_details"][] = [
                    "order_item_detail_id" => $orderItemId,
                    "name"                 => $buyer['name'] ?? null,
                    "phone"                => $buyer['phone'] ?? null,
                ];

                // --- fks_buyer_addresses
                if (!empty($buyer['address'])) {
                    $address = $buyer['address'];
                    $result["fks_buyer_addresses"][] = [
                        "buyer_detail_id" => $buyerId,
                        "line1"           => $address['line1'] ?? null,
                        "line2"           => $address['line2'] ?? null,
                        "city"            => $address['city'] ?? null,
                        "pincode"         => $address['pincode'] ?? null,
                        "state"           => $address['state'] ?? null,
                    ];
                }
            }

            // --- fks_offers
            if (!empty($detail['offers'])) {
                foreach ($detail['offers'] as $offer) {
                    $result["fks_offers"][] = [
                        "shipment_order_item_id" => $orderItemId,
                        "offerId"                => $offer['offerId'] ?? null,
                        "adjustmentType"         => $offer['adjustmentType'] ?? null,
                        "isSellerPromotion"      => $offer['isSellerPromotion'] ?? null,
                        "offerDescription"       => $offer['offerDescription'] ?? null,
                        "offerValueInRupees"     => $offer['offerValueInRupees'] ?? null,
                        "promotionId"            => $offer['promotionId'] ?? null,
                        "sellerContribution"     => $offer['sellerContribution'] ?? null,
                    ];
                }
            }

            // --- fks_imei_details
            if (!empty($item['imeiDetails']['imei'])) {
                foreach ($item['imeiDetails']['imei'] as $imei) {
                    if (!empty($imei)) {
                        $result["fks_imei_details"][] = [
                            "shipment_order_item_id" => $orderItemId,
                            "imei"                   => $imei,
                        ];
                    }
                }
            }
        }

        // --- fks_sub_shipments + fks_packages
        if (!empty($shipment['subShipments'])) {
            foreach ($shipment['subShipments'] as $sub) {
                $subId = $sub['externalSubShipmentId'];
                $result["fks_sub_shipments"][] = [
                    "shipment_id"          => $shipmentId,
                    "externalSubShipmentId"=> $sub['externalSubShipmentId'],
                ];

                if (!empty($sub['packages'])) {
                    foreach ($sub['packages'] as $pkg) {
                        $result["fks_packages"][] = [
                            "sub_shipment_id" => $subId,
                            "packageId"       => $pkg['packageId'],
                            "length"          => $pkg['dimensions']['length'] ?? null,
                            "breadth"         => $pkg['dimensions']['breadth'] ?? null,
                            "height"          => $pkg['dimensions']['height'] ?? null,
                            "weight"          => $pkg['dimensions']['weight'] ?? null,
                        ];
                    }
                }
            }
        }
    }

    return $result;
}

// Example usage:
$json = file_get_contents("shipments.json");
$data = json_decode($json, true);
$parsed = parseFlipkartShipments($data);

print_r($parsed); // or pass to Laravel models for import
