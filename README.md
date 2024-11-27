# midocean-woo-product-import Documentation

## Midocean API explanation

1. `https://api.midocean.com/gateway/products/2.0?language=es` this is the endpoint to get all products. on my plugin i integrate in `inc/files/file-insert-products-db.php` file. actually all of the endpoint i integrate in this file.
2. ` https://api.midocean.com/gateway/stock/2.0` this endpoint for get product stock. the stocks update very hour. i create a custom endpoint to get stock from this api and insert them into database. after while update the products every minute the stock will update.
3. ` https://api.midocean.com/gateway/pricelist/2.0/` this endpoint to get product price. this the base price of products. this is update daily. i create a custom endpoint to get price from this api and insert them into database.
4. ` https://api.midocean.com/gateway/printpricelist/2.0/` this endpoint to get product print price. this the print price of products. this is update daily. when select a printing position the print price will calculate based on these price. [Screenshot](https://prnt.sc/-lrZIBPavms8)
5. ` https://api.midocean.com/gateway/printdata/1.0` this endpoint to get product print data. Printing options that are available for the
   midocean products, print positions, available printing techniques per position, the max print size, max print colors and template images of the positions. [screenshot](https://prnt.sc/OK9pvIvdhHTl)
6. ` https://api.midocean.com/gateway/order/2.1/create` the is the endpoint to create order. there are three types of order `SAMPLE` `NORMAL` `PRINT`. it will decide based on select printing position. if nothing select printing position it will be a `NORMAL order. if select it will be PRINT order.

## Fetch products and insert them into database

```php

// fetch products from api
function fetch_products_from_api() {

   // get api key
   $api_key = get_option( 'be-api-key' ) ?? '';

   $curl = curl_init();
   curl_setopt_array(
       $curl,
       array(
           CURLOPT_URL            => 'https://api.midocean.com/gateway/products/2.0?language=es',
           CURLOPT_RETURNTRANSFER => true,
           CURLOPT_ENCODING       => '',
           CURLOPT_MAXREDIRS      => 10,
           CURLOPT_TIMEOUT        => 0,
           CURLOPT_FOLLOWLOCATION => true,
           CURLOPT_HTTP_VERSION   => CURL_HTTP_VERSION_1_1,
           CURLOPT_CUSTOMREQUEST  => 'GET',
           CURLOPT_HTTPHEADER     => array(
               'x-Gateway-APIKey: ' . $api_key,
           ),
       )
   );

   $response = curl_exec( $curl );

   curl_close( $curl );
   return $response;
}

// insert products to database
function insert_products_db() {

   // Fetch api response
   $api_response = fetch_products_from_api();

   // Decode to array
   $products = json_decode( $api_response, true );

   // Insert to database
   global $wpdb;
   $table_prefix   = get_option( 'be-table-prefix' ) ?? '';
   $products_table = $wpdb->prefix . $table_prefix . 'sync_products';
   truncate_table( $products_table );

   foreach ( $products as $product ) {

       // Extract product variants
       $product_variants = $product['variants'];
       $product_number   = '';

       // Loop through variants to get product number/sku
       foreach ( $product_variants as $variant ) {
           $product_number = $variant['sku'];
       }

       // extract products
       $product_data = json_encode( $product );

       $wpdb->insert(
           $products_table,
           [
               'product_number' => $product_number,
               'product_data'   => $product_data,
               'status'         => 'pending',
           ]
       );
   }

   return '<h4>Products inserted successfully DB</h4>';
}

```

this provided codes snippets first fetch products from midocean api and insert them into database. i integrate others endpoints such as stock, price, print price, print data in this file. for more details on about these endpoint see the midocean api documentations.

**before insert to database i need to create the tables first** i create all of required tables in `inc/files/file-db-table-create.php` file.

## Database Table creation

```php

function sync_products() {
   global $wpdb;

   $table_prefix    = get_option( 'be-table-prefix' ) ?? '';
   $table_name      = $wpdb->prefix . $table_prefix . 'sync_products';
   $charset_collate = $wpdb->get_charset_collate();

   $sql = "CREATE TABLE IF NOT EXISTS $table_name (
       id INT AUTO_INCREMENT,
       product_number VARCHAR(255) NULL,
       product_data LONGTEXT NOT NULL,
       status VARCHAR(255) NOT NULL,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
       PRIMARY KEY (id)
   ) $charset_collate;";

   require_once ABSPATH . 'wp-admin/includes/upgrade.php';
   dbDelta( $sql );
}

// Remove sync_products Table when plugin deactivated
function remove_sync_products() {
   global $wpdb;

   $table_prefix = get_option( 'be-table-prefix' ) ?? '';
   $table_name   = $wpdb->prefix . $table_prefix . 'sync_products';
   $sql          = "DROP TABLE IF EXISTS $table_name;";
   $wpdb->query( $sql );
}

function create_db_tables() {
   sync_products();
   sync_stock();
   sync_price();
   sync_products_print_data();
   sync_print_price();
   // sync_products_print_data_labels();
   sync_color_group();
   sync_color_hex_list();
}

function remove_db_tables() {
   remove_sync_products();
   remove_sync_stock();
   remove_sync_price();
   remove_products_print_data();
   remove_sync_print_price();
   // remove_products_print_data_labels();
   remove_color_group();
   remove_color_hex_list();
}

```

this provided codes snippets i create all of required tables in database. same as others in this file.

The tables will created when plugin activated and removed when plugin deactivated.

## Product create on WooCommerce

for crate products on woocommerce i write codes on `inc/files/file-import-products-woo.php` file.

#### Workflows

1. Define the tables variable
2. Connect with the woocommerce store credentials. get from `WooCommerce>Settings>Advanced>REST API`
3. Get product from database with joining with product, stock, price tables.
4. Extract the necessary data.
5. Create product on woocommerce store.

```php

$sql = "SELECT sp.id, sp.product_number, sp.product_data, ss.stock, spr.variant_id, spr.price, spr.valid_until, sp.status  FROM $products_table sp JOIN $stock_table ss ON sp.product_number = ss.product_number JOIN $price_table spr ON sp.product_number = spr.product_number WHERE sp.status = 'pending' LIMIT 1";

```

This is the sql query to get product from database with joining. for more details see `inc/files/file-import-products-woo.php` file codes.

## Single product page configuration and Product Configuration

[screenshot](https://prnt.sc/YY1MmvCSFom4)
[screenshot](https://prnt.sc/0AavOeYk9dQm)

the single page configuration and product confutator codes written in `inc/classes/class-customize-product-page.php` file. I Create some shortcode for indibusal use. the **display_product_info** shortcode to display product info into accordion. [screenshot](https://prnt.sc/YunuiPN5__eb). the **display_product_sku** shortcode to display the product sku with expected design [screenshot](https://prnt.sc/UZUEZkPAnSk0). the **custom_product_configurator_mto_link** shortcode to display mto link [screenshot](https://prnt.sc/r4KdU5H3kQz1). the **custom_product_configurator** shortcode to display product configurator [screenshot](https://prnt.sc/QMxqhiaPyj9L).

## Directory and File Structure

`**assets**` directory

this is the directory that contains all of required assets such as css, js, images.

`**inc**` directory

this is the directory that contains all of required files. it has sub directory.

`inc/classes` directory

this is the directory that contains all of required classes. `class-autoloader.php` file for load all classes in this directory. the `class-admin-menu` file for create admin menu [screenshot](https://prnt.sc/Xhs2MTU83RSu). the `class-api-endpoints.php` file for commonly used api endpoints. the `class-create-order.php` file for integrate order api. when place an order if selected the printing positions it will place order to midocean site PRINT order otherwise NORMAL order. the `class-customize-product-page.php` file for customize the single product page. the `class-display-additional-info.php` file for display additional information's into additional info tab. the `class-display-additional-info.php` file for enqueue asset like admin/public css, admin/public js. the `class-template.php` is the demo class template. when create a new class in the inc/classes directory copy the template and start writing codes. after that instantiate the class into class-autoloader.php file.

```php

<?php
/**
 * Bootstraps the plugin.
 */

namespace BULK_IMPORT\Inc;

defined( "ABSPATH" ) || exit( "Direct Access Not Allowed" );

use BULK_IMPORT\Inc\Traits\Singleton;

class Autoloader {
    use Singleton;

    protected function __construct() {

        // load class.
        Template::get_instance();
    }
}

```

it will auto load the class.

`inc/files` directory

the `color-group.json` file is the transformed color group list. the `color-name.json` file is the color hex with color id. it was transformed from `/uploads/color-list.json` file. the `file-api-endpoints.php` file is the endpoints created by me. [screenshot](https://prnt.sc/UGl4si5pXFf2). the `file-db-table-create.php` file for create database tables. the `file-import-products-woo.php` for insert/sync products to woocommerce. the `file-insert-products-db.php` for fetch apis data and insert them into database. the `file-transform-color-list.php` for transform colors from raw color list into my expected format. the `labels.json` file store printing position labels [screenshot](https://prnt.sc/vrWW6eE7p1vO). the `manipulation-cost.json` file store the manipulation cost. it get from product print price data.

`inc/helpers` directory

the helper directory for commonly used helper functions.

`inc/template-parts` directory

the template directory for template parts. like template-api for api endpoints template [screenshot](https://prnt.sc/InzzIyP6LRFO) same as others.

`inc/traits` directory

the trait directory for trait classes. like trait-singleton for singleton trait.

## Order api integration

in `inc/classes/class-create-order.php` file for integrate order api.

```php

public function setup_hooks() {
        // Add a WooCommerce action hook that triggers on the thank you page
        add_action( 'woocommerce_thankyou', [ $this, 'create_order' ] );
        add_action( 'woocommerce_thankyou', [ $this, 'display_printing_positions_on_thank_you_page' ], 20, 1 );
        add_action( 'woocommerce_order_details_after_order_table', [ $this, 'display_printing_positions_on_order_page' ], 20, 1 );
        add_action( 'woocommerce_before_calculate_totals', [ $this, 'update_cart_item_price' ], 10, 1 );
    }

```

the `woocommerce_thankyou` hook trigger when a order complete. after successfully place an order it will extract all of order information's and order data and will be manipulate based on the midocean order api payload and place an order to midocean. after successfully create order the extra information's will be display on thankyou page and order page.