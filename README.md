PayPal Payment Gateway Integration in PHP
=========================================

by [Vincy](https://phppot.com/about/). Last modified on September 1st, 2020.

This tutorial will help you to integrate PayPal payment gateway using PHP with an example. I will walk you through a step by step process and make it easy for you do the integration.

You will learn about three things today,

1.  How to integrate PayPal payment gateway in your application and process payments.
2.  What is PayPal IPN, why you should use it and an example implementation.
3.  How to test PayPal payment gateway integration using PayPal sandbox.

PayPal provides different options to integrate the gateway like,

1.  PayPal Payments Standard
2.  Checkout
3.  PayPal Commerce Platform
4.  [Recurring payments using PayPal Subscriptions](https://phppot.com/php/how-to-manage-recurring-payments-using-paypal-subscriptions-in-php/)
5.  Payouts
6.  Invoicing
7.  Accept card payments
8.  PayPal Pro

In this example, we will use "PayPal Payments Standard" method for integrating the payment gateway.

What is inside?
---------------

1.  How it works?
2.  Payment gateway integration steps
3.  File Structure
4.  About PHP example to integrate payment using PayPal Payments Standard
5.  Database Script
6.  Go live
7.  Payment gateway integration output

How it works?
-------------

![payment_gateway_integration](https://phppot.com/wp-content/uploads/2015/08/payment_gateway_integration.png)

PayPal payment gateway integration steps
----------------------------------------

1.  Render PayPal payment form HTML with required parameters and "custom Pay Now button".
2.  Set the suitable values for the form fields to fill up item information, merchant email id, callback URLs.
3.  Set suitable form action with PayPal sandbox/live URL endpoint to which the payment details has to be posted on submit.
4.  Handle payment response in IPN listener to update database or to send email to the customer or both.
5.  Acknowledge user appropriately with the return/cancel page after payment cycle.

If you wish to process card payments refer the [Stripe payment gateway integration using PHP](https://phppot.com/php/stripe-payment-gateway-integration-using-php/) articles.

File structure
--------------

Below screenshot shows the file structure of this PHP example code created for integrating Payment Gateway using PayPal Payments Standard method.

![PayPal Payment Integration Example File Structure](https://phppot.com/wp-content/uploads/2015/08/paypal-payment-integration-example-file-structure.jpg)

The landing page shows an entry point for proceeding with the PayPal payment transaction process. It will show a sample product tile containing PayPal payment fields following by a Pay Now button.

The return.php and the cancel.php files are the acknowledgement pages to which the user will be redirected appropriately. The notify.php is the IPN listener connect database to save payment transaction response and data.

DBController.php file contains the code to create database connection object and to handle queries on payment database update.

About PHP example to integrate payment using PayPal Payments Standard
---------------------------------------------------------------------

### PayPal Payments Standard form HTML

In this HTML code we can see the PayPal Payments Standard form fields with a custom Pay Now button. This example code targets the PayPal sandbox URL on the form action.

The PayPal payment form must contain the business email address, order information like item_number, name, amount, currency, and the callback and return page URLs.

By submitting this form, the payment parameters will be posted to the PayPal server. So, the user will be redirected to the PayPal Login and asked for payment confirmation.

```html
<body>
    <div id="payment-box">
        <img src="images/camera.jpg" />
        <h4 class="txt-title">A6900 MirrorLess Camera</h4>
        <div class="txt-price">$289.61</div>
        <form action="https://www.sandbox.paypal.com/cgi-bin/webscr"
            method="post" target="_top">
            <input type='hidden' name='business'
                value='PayPal Business Email'> <input type='hidden'
                name='item_name' value='Camera'> <input type='hidden'
                name='item_number' value='CAM#N1'> <input type='hidden'
                name='amount' value='10'> <input type='hidden'
                name='no_shipping' value='1'> <input type='hidden'
                name='currency_code' value='USD'> <input type='hidden'
                name='notify_url'
                value='http://sitename/paypal-payment-gateway-integration-in-php/notify.php'>
            <input type='hidden' name='cancel_return'
                value='http://sitename/paypal-payment-gateway-integration-in-php/cancel.php'>
            <input type='hidden' name='return'
                value='http://sitename/paypal-payment-gateway-integration-in-php/return.php'>
            <input type="hidden" name="cmd" value="_xclick"> <input
                type="submit" name="pay_now" id="pay_now"
                Value="Pay Now">
        </form>
    </div>
</body>
```

PayPal Notifications
--------------------

PayPal provides  notifications using different mechanisms. It will be useful for updating your backend processes, sending order notification and similar transactional jobs. Following are the different types of notifications provided by PayPal.

1.  Webhooks
2.  IPN
3.  PDT

Notifications should be used as part of the payment gateway integration process. Imagine you are running a website that sells digital goods online. At the end of the payment process, you are obliged to send the digital product via email.

You should not do this in the thank-you page. As there is no guarantee that this page will be displayed in the user's browser. There can be network failures, user's might close the browser and there are so many variables involved in this.

The dependable way of doing this is using PayPal notifications. You get a callback from PayPal to your backend server (asynchronously) and you can manage the backend process from there.

### What is PayPal IPN?

PayPal notifies the merchants about the events related to their transactions. This automatic callback can be used to perform administrative tasks and to fulfil orders. 

Notifications are done by PayPal on different type of events like, payments received, authorizations made, recurring payments, subscriptions, chargebacks, disputes, reversals and refunds.

An integral part of the payment gateway integration process in the ability to receive PayPal notification and process backend administrative processes.

The order fulfillment is an important step in a [shopping cart software](https://phppot.com/php/simple-php-shopping-cart/). It should be done via the PayPal notification and never should be done as part of the thank you or order completion page.

### Instant Payment Notification listener

The notify.php is created as the instant payment notification listener. This listener URL will be sent to PayPal via the form seen above. It can also be set in the PayPal REST API app settings. Paypal provides the [IPN listener code samples](https://github.com/paypal/ipn-code-samples) for various languages.

This listener will be called asynchronously by PayPal to notify payment processing response. The payment response will be posted to this URL.

Below code verifies the payment status from the response returned by PayPal. If the payment is verified and completed then the result will be updated in a database table.

```php 
<?php
// CONFIG: Enable debug mode. This means we'll log requests into 'ipn.log' in the same directory.
// Especially useful if you encounter network errors or other intermittent problems with IPN (validation).
// Set this to 0 once you go live or don't require logging.
define("DEBUG", 1);
// Set to 0 once you're ready to go live
define("USE_SANDBOX", 1);
define("LOG_FILE", "ipn.log");
// Read POST data
// reading posted data directly from $_POST causes serialization
// issues with array data in POST. Reading raw POST data from input stream instead.
$raw_post_data = file_get_contents('php://input');
$raw_post_array = explode('&', $raw_post_data);
$myPost = array();
foreach ($raw_post_array as $keyval) {
	$keyval = explode ('=', $keyval);
	if (count($keyval) == 2)
		$myPost[$keyval[0]] = urldecode($keyval[1]);
}
// read the post from PayPal system and add 'cmd'
$req = 'cmd=_notify-validate';
if(function_exists('get_magic_quotes_gpc')) {
	$get_magic_quotes_exists = true;
}
foreach ($myPost as $key => $value) {
	if($get_magic_quotes_exists == true && get_magic_quotes_gpc() == 1) {
		$value = urlencode(stripslashes($value));
	} else {
		$value = urlencode($value);
	}
	$req .= "&$key=$value";
}
// Post IPN data back to PayPal to validate the IPN data is genuine
// Without this step anyone can fake IPN data
if(USE_SANDBOX == true) {
	$paypal_url = "https://www.sandbox.paypal.com/cgi-bin/webscr";
} else {
	$paypal_url = "https://www.paypal.com/cgi-bin/webscr";
}
$ch = curl_init($paypal_url);
if ($ch == FALSE) {
	return FALSE;
}
curl_setopt($ch, CURLOPT_HTTP_VERSION, CURL_HTTP_VERSION_1_1);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_RETURNTRANSFER,1);
curl_setopt($ch, CURLOPT_POSTFIELDS, $req);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 1);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);
curl_setopt($ch, CURLOPT_FORBID_REUSE, 1);
if(DEBUG == true) {
	curl_setopt($ch, CURLOPT_HEADER, 1);
	curl_setopt($ch, CURLINFO_HEADER_OUT, 1);
}
// CONFIG: Optional proxy configuration
//curl_setopt($ch, CURLOPT_PROXY, $proxy);
//curl_setopt($ch, CURLOPT_HTTPPROXYTUNNEL, 1);
// Set TCP timeout to 30 seconds
curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 30);
curl_setopt($ch, CURLOPT_HTTPHEADER, array('Connection: Close'));
// CONFIG: Please download 'cacert.pem' from "http://curl.haxx.se/docs/caextract.html" and set the directory path
// of the certificate as shown below. Ensure the file is readable by the webserver.
// This is mandatory for some environments.
//$cert = __DIR__ . "./cacert.pem";
//curl_setopt($ch, CURLOPT_CAINFO, $cert);
$res = curl_exec($ch);
if (curl_errno($ch) != 0) // cURL error
	{
	if(DEBUG == true) {	
		error_log(date('[Y-m-d H:i e] '). "Can't connect to PayPal to validate IPN message: " . curl_error($ch) . PHP_EOL, 3, LOG_FILE);
	}
	curl_close($ch);
	exit;
} else {
		// Log the entire HTTP response if debug is switched on.
		if(DEBUG == true) {
			error_log(date('[Y-m-d H:i e] '). "HTTP request of validation request:". curl_getinfo($ch, CURLINFO_HEADER_OUT) ." for IPN payload: $req" . PHP_EOL, 3, LOG_FILE);
			error_log(date('[Y-m-d H:i e] '). "HTTP response of validation request: $res" . PHP_EOL, 3, LOG_FILE);
		}
		curl_close($ch);
}
// Inspect IPN validation result and act accordingly
// Split response headers and payload, a better way for strcmp
$tokens = explode("\r\n\r\n", trim($res));
$res = trim(end($tokens));
if (strcmp ($res, "VERIFIED") == 0) {
	// assign posted variables to local variables
	$item_name = $_POST['item_name'];
	$item_number = $_POST['item_number'];
	$payment_status = $_POST['payment_status'];
	$payment_amount = $_POST['mc_gross'];
	$payment_currency = $_POST['mc_currency'];
	$txn_id = $_POST['txn_id'];
	$receiver_email = $_POST['receiver_email'];
	$payer_email = $_POST['payer_email'];

	include("DBController.php");
	$db = new DBController();

	// check whether the payment_status is Completed
	$isPaymentCompleted = false;
	if($payment_status == "Completed") {
		$isPaymentCompleted = true;
	}
	// check that txn_id has not been previously processed
	$isUniqueTxnId = false;
	$param_type="s";
	$param_value_array = array($txn_id);
	$result = $db->runQuery("SELECT * FROM payment WHERE txn_id = ?",$param_type,$param_value_array);
	if(empty($result)) {
        $isUniqueTxnId = true;
	}	
	// check that receiver_email is your PayPal email
	// check that payment_amount/payment_currency are correct
	if($isPaymentCompleted) {
	    $param_type = "sssdss";
	    $param_value_array = array($item_number, $item_name, $payment_status, $payment_amount, $payment_currency, $txn_id);
	    $payment_id = $db->insert("INSERT INTO payment(item_number, item_name, payment_status, payment_amount, payment_currency, txn_id) VALUES(?, ?, ?, ?, ?, ?)", $param_type, $param_value_array);

	}
	// process payment and mark item as paid.

	if(DEBUG == true) {
		error_log(date('[Y-m-d H:i e] '). "Verified IPN: $req ". PHP_EOL, 3, LOG_FILE);
	}

} else if (strcmp ($res, "INVALID") == 0) {
	// log for manual investigation
	// Add business logic here which deals with invalid IPN messages
	if(DEBUG == true) {
		error_log(date('[Y-m-d H:i e] '). "Invalid IPN: $req" . PHP_EOL, 3, LOG_FILE);
	}
}
?>
``` 

### DBController.php

This file handles database related functions using prepared statement with MySQLi. This is important, always remember to use Prepared Statements as it will safeguard you from SQL injections and a primary step towards your security of the application.

```php
<?php

class DBController {

    private $host = "";

    private $user = "";

    private $password = "";

    private $database = "";

    private $conn;

    function __construct() {
        $this->conn = $this->connectDB();
    }

    function connectDB() {
        $conn = mysqli_connect($this->host, $this->user, $this->password, $this->database);
        return $conn;
    }

    function runQuery($query, $param_type, $param_value_array) {
        $sql = $this->conn->prepare($query);
        $this->bindQueryParams($sql, $param_type, $param_value_array);
        $sql->execute();
        $result = $sql->get_result();

        if ($result->num_rows > 0) {
            while ($row = $result->fetch_assoc()) {
                $resultset[] = $row;
            }
        }

        if (! empty($resultset)) {
            return $resultset;
        }
    }

    function bindQueryParams($sql, $param_type, $param_value_array) {
        $param_value_reference[] = & $param_type;
        for ($i = 0; $i < count($param_value_array); $i ++) {
            $param_value_reference[] = & $param_value_array[$i];
        }
        call_user_func_array(array(
            $sql,
            'bind_param'
        ), $param_value_reference);
    }

    function insert($query, $param_type, $param_value_array) {
        $sql = $this->conn->prepare($query);
        $this->bindQueryParams($sql, $param_type, $param_value_array);
        $sql->execute();
    }
}
?>
``` 

Database script
---------------

This section shows the payment database table structure in .sql format. This is provided with the downloadable source code with sql/payment.sql file. Import this file in your test environment to run this example.

--
-- Table structure for table `payment`
--

```sql
CREATE TABLE IF NOT EXISTS `payment` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `item_number` varchar(255) NOT NULL,
  `item_name` varchar(255) NOT NULL,
  `payment_status` varchar(255) NOT NULL,
  `payment_amount` double(10,2) NOT NULL,
  `payment_currency` varchar(255) NOT NULL,
  `txn_id` varchar(255) NOT NULL,
  `create_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
);
``` 

How to test PayPal payment gateway integration using sandbox
------------------------------------------------------------

Test! Test! Test! It is important. We are processing payments and you cannot be casual about this implementation. Every bit of your PHP script should be tested thoroughly before going live.

PayPal provides excellent toolset to perform the end to end testing of the payment gateway integration. It provides a sandbox environment using which you can simulate the complete payment process including the payments and fulfillments step.

Testing the IPN is critical as it takes care of the order fulfilment process. Go through the below steps to setup a PayPal Sandbox account and test you PHP code for payment gateway integration.

1.  *Sign Up* with the PayPal developer account <https://developer.paypal.com/>
2.  Create sandbox Business and Personal accounts via *Dashboard -> Sandbox -> Accounts*.
3.  Specify the sandbox business account email address in the payment form.
4.  Add PayPal sandbox endpoint URL https://www.sandbox.paypal.com/cgi-bin/webscr to the payment form action.
5.  If you are using IPN, add sandbox URL to access via cURL.

Go live
-------

Once everything is working good with the entire payment transaction processing flow, then it will be a time to go live with your PayPal payment gateway integration using PHP code. Change the sandbox mode to live mode and do the following.

It requires to do the following changes in the code.

1.  Replace the sandbox business account email with live email address.

    <input type='hidden' name='business' value='PayPal Business Email'>

2.  Change the PayPal Payments Standard form action URL as shown below.

    <form action="https://www.paypal.com/cgi-bin/webscr" method="post" target="_top">

3.  In notify.php, set the USE_SANDBOX constant to 0.

    define("USE_SANDBOX", 1);

Payment gateway integration output
----------------------------------

Below screenshot shows the product tile which includes PayPal payments standard form and custom Pay Now button.

![Payment Integration Landing Page Output](https://phppot.com/wp-content/uploads/2015/08/payment-integration-landing-page-output.jpg)

After successful payment transaction, the user will be automatically redirected to the return URL as specified in the payment form. The return page will acknowledge the user as shown below.

The return url is nothing but a thank you page or order completion page. This is the last step in the payment gateway integration process.

An important thing to know is that the order fulfilment steps should not be done sequentially in the return URL call. It should be done via the IPN callback or the other PayPal notifications.