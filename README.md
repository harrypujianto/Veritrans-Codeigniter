Veritrans Codeigniter library
=======================================================
Veritrans :heart: CI!

This is the all new Codeigniter client library for Veritrans 2.0. Visit [https://www.veritrans.co.id](https://www.veritrans.co.id) for more information about the product and see documentation at [http://docs.veritrans.co.id](http://docs.veritrans.co.id) for more technical details.

### Requirements
The following plugin is tested under following environment:
* PHP v5.4.x or greater
* Codeigniter v2.2.x

## Installation
* Download the library and extract the .zip 
* Merge all the files to your codeigniter directory

## Using Veritrans Library

### load library
```php
//set production to true for production environment
$params = array('server_key' => '<your server key>', 'production' => false);
$this->load->library('veritrans');
$this->veritrans->config($params);
```

### VT-Web

You can see some VT-Web examples [here](https://github.com/harrypujianto/Veritrans-Codeigniter/blob/master/application/controllers/vtweb.php).

#### Get Redirection URL of a Charge
```php
//you don't have to use the function name 'vtweb_charge', it's just an example
public function vtweb_charge()
{
    $params = array(
        'transaction_details' => array(
          'order_id' => rand(),
          'gross_amount' => 10000,
        ),
        'vtweb' => array()
      );
    
    try {
      // Redirect to Veritrans VTWeb page
      header('Location: ' . $this->veritrans->vtweb_charge($params));
    }
    catch (Exception $e) {
      echo $e->getMessage();
    }
}
```

#### Handle Notification Callback
You can see notification handler examples [here](https://github.com/harrypujianto/Veritrans-Codeigniter/blob/master/application/controllers/vtweb.php).
```php
//you don't have to use the function name 'notification', it's just an example
public function notificarion()
{
        $json_result = file_get_contents('php://input');
		$result = json_decode($json_result);
		if($result){
		$notif = $this->veritrans->status($result->order_id);
		}
		
		$transaction = $notif->transaction_status;
		$type = $notif->payment_type;
		$order_id = $notif->order_id;
		$fraud = $notif->fraud_status;
		if ($transaction == 'capture') {
		  // For credit card transaction, we need to check whether transaction is challenge by FDS or not
		  if ($type == 'credit_card'){
		    if($fraud == 'challenge'){
		      // TODO set payment status in merchant's database to 'Challenge by FDS'
		      // TODO merchant should decide whether this transaction is authorized or not in MAP
		      echo "Transaction order_id: " . $order_id ." is challenged by FDS";
		      } 
		      else {
		      // TODO set payment status in merchant's database to 'Success'
		      echo "Transaction order_id: " . $order_id ." successfully captured using " . $type;
		      }
		    }
		  }
		else if ($transaction == 'settlement' && $type != 'credit_card'){
		  // TODO set payment status in merchant's database to 'Settlement'
		  echo "Transaction order_id: " . $order_id ." successfully transfered using " . $type;
		  } 
		  else if($transaction == 'pending'){
		  // TODO set payment status in merchant's database to 'Pending'
		  echo "Waiting customer to finish transaction order_id: " . $order_id . " using " . $type;
		  } 
		  else if ($transaction == 'deny') {
		  // TODO set payment status in merchant's database to 'Denied'
		  echo "Payment using " . $type . " for transaction order_id: " . $order_id . " is denied.";
		}
```

### VT-Direct
You can see VT-Direct form [here](https://github.com/harrypujianto/Veritrans-Codeigniter/blob/master/application/views/checkout_with_3ds.php).

you can see VT-Direct process [here](https://github.com/harrypujianto/Veritrans-Codeigniter/blob/master/application/controllers/vtdirect.php).

#### Checkout Page

```html
<html>
<head>
	<title>Checkout</title>
	<!-- Include PaymentAPI  -->
	<link rel="stylesheet" href="<?php echo base_url();?>asset/css/jquery.fancybox.css">
</head>
<body>
	<script type="text/javascript" src="https://api.sandbox.veritrans.co.id/v2/assets/js/veritrans.min.js"></script>
	<script src="//ajax.googleapis.com/ajax/libs/jquery/1.11.0/jquery.min.js"></script>
	<script type="text/javascript" src="<?php echo base_url();?>asset/js/jquery.fancybox.pack.js"></script>

	<h1>Checkout</h1>
	<form action="<?php echo site_url()?>/vtdirect/vtdirect_cc_charge" method="POST" id="payment-form">
		<fieldset>
			<legend>Checkout</legend>
			<p>
				<label>Card Number</label>
				<input class="card-number" value="4811111111111114" size="20" type="text" autocomplete="off"/>
			</p>
			<p>
				<label>Expiration (MM/YYYY)</label>
				<input class="card-expiry-month" value="12" placeholder="MM" size="2" type="text" />
		    	<span> / </span>
		    	<input class="card-expiry-year" value="2018" placeholder="YYYY" size="4" type="text" />
			</p>
			<p>
		    	<label>CVV</label>
		    	<input class="card-cvv" value="123" size="4" type="password" autocomplete="off"/>
			</p>

			<p>
		    	<label>Save credit card</label>
		    	<input type="checkbox" name="save_cc" value="true">
			</p>

			<input id="token_id" name="token_id" type="hidden" />
			<button class="submit-button" type="submit">Submit Payment</button>
		</fieldset>
	</form>

	<!-- Javascript for token generation -->
	<script type="text/javascript">
	$(function(){
		// Sandbox URL
		Veritrans.url = "https://api.sandbox.veritrans.co.id/v2/token";
		// TODO: Change with your client key.
		Veritrans.client_key = "<your client key>";
		
		//Veritrans.client_key = "d4b273bc-201c-42ae-8a35-c9bf48c1152b";
		var card = function(){
			return { 	'card_number'		: $(".card-number").val(),
						'card_exp_month'	: $(".card-expiry-month").val(),
						'card_exp_year'		: $(".card-expiry-year").val(),
						'card_cvv'			: $(".card-cvv").val(),
						'secure'			: true,
						'bank'				: 'bni',
						'gross_amount'		: 10000
						 }
		};

		function callback(response) {
			if (response.redirect_url) {
				// 3dsecure transaction, please open this popup
				openDialog(response.redirect_url);

			} else if (response.status_code == '200') {
				// success 3d secure or success normal
				closeDialog();
				// submit form
				$(".submit-button").attr("disabled", "disabled"); 
				$("#token_id").val(response.token_id);
				$("#payment-form").submit();
			} else {
				// failed request token
				console.log('Close Dialog - failed');
				//closeDialog();
				//$('#purchase').removeAttr('disabled');
				// $('#message').show(FADE_DELAY);
				// $('#message').text(response.status_message);
				//alert(response.status_message);
			}
		}

		function openDialog(url) {
			$.fancybox.open({
		        href: url,
		        type: 'iframe',
		        autoSize: false,
		        width: 700,
		        height: 500,
		        closeBtn: false,
		        modal: true
		    });
		}

		function closeDialog() {
			$.fancybox.close();
		}
		
		$('.submit-button').click(function(event){
			event.preventDefault();
			//$(this).attr("disabled", "disabled"); 
			Veritrans.token(card, callback);
			return false;
		});
	});

	</script>
</body>
</html>
```

#### Checkout Process

##### 1. Create Transaction Details

```php
$transaction_details = array(
  'order_id'    => time(),
  'gross_amount'  => 10000
);
```

##### 2. Create Item Details, Billing Address, Shipping Address, and Customer Details (Optional)

```php
// Populate items
$items = array(
    array(
      'id'       => 'item1',
      'price'    => 5000,
      'quantity' => 1,
      'name'     => 'Adidas f50'
    ),
    array(
      'id'       => 'item2',
      'price'    => 2500,
      'quantity' => 2,
      'name'     => 'Nike N90'
    ));

// Populate customer's billing address
$billing_address = array(
    'first_name'   => "Andri",
    'last_name'    => "Setiawan",
    'address'      => "Karet Belakang 15A, Setiabudi.",
    'city'         => "Jakarta",
    'postal_code'  => "51161",
    'phone'        => "081322311801",
    'country_code' => 'IDN'
  );

// Populate customer's shipping address
$shipping_address = array(
    'first_name'   => "John",
    'last_name'    => "Watson",
    'address'      => "Bakerstreet 221B.",
    'city'         => "Jakarta",
    'postal_code'  => "51162",
    'phone'        => "081322311801",
    'country_code' => 'IDN'
  );

// Populate customer's info
$customer_details = array(
    'first_name'       => "Andri",
    'last_name'        => "Setiawan",
    'email'            => "andri@email.co",
    'phone'            => "081322311801",
    'billing_address'  => $billing_address,
    'shipping_address' => $shipping_address
  );
```

##### 3. Get Token ID from Checkout Page

```php
// Token ID from checkout page
$token_id = $_POST['token_id'];
```
##### 4. Create Transaction Data

```php
// Transaction data to be sent
$transaction_data = array(
    'payment_type' => 'credit_card',
    'credit_card'  => array(
      'token_id'      => $token_id,
      'bank'          => 'bni',
      'save_token_id' => isset($_POST['save_cc'])
    ),
    'transaction_details' => $transaction_details,
    'item_details'        => $items,
    'customer_details'    => $customer_details
  );
```

##### 5. Charge

```php
$response= $this->veritrans->vtdirect_charge($transaction_data);
```

##### 6. Handle Transaction Status

```php
// Success
if($response->transaction_status == 'capture') {
  echo "<p>Transaksi berhasil.</p>";
  echo "<p>Status transaksi untuk order id $response->order_id: " .
      "$response->transaction_status</p>";

  echo "<h3>Detail transaksi:</h3>";
  echo "<pre>";
  var_dump($response);
  echo "</pre>";
}
// Deny
else if($response->transaction_status == 'deny') {
  echo "<p>Transaksi ditolak.</p>";
  echo "<p>Status transaksi untuk order id .$response->order_id: " .
      "$response->transaction_status</p>";

  echo "<h3>Detail transaksi:</h3>";
  echo "<pre>";
  var_dump($response);
  echo "</pre>";
}
// Challenge
else if($response->transaction_status == 'challenge') {
  echo "<p>Transaksi challenge.</p>";
  echo "<p>Status transaksi untuk order id $response->order_id: " .
      "$response->transaction_status</p>";

  echo "<h3>Detail transaksi:</h3>";
  echo "<pre>";
  var_dump($response);
  echo "</pre>";
}
// Error
else {
  echo "<p>Terjadi kesalahan pada data transaksi yang dikirim.</p>";
  echo "<p>Status message: [$response->status_code] " .
      "$response->status_message</p>";

  echo "<pre>";
  var_dump($response);
  echo "</pre>";
}
```

#### Process Transaction

##### Get a Transaction Status

```php
$status = $this->veritrans->status($order_id);
var_dump($status);
```
##### Approve a Transaction

```php
$approve = $this->veritrans->approve($order_id);
var_dump($approve);
```

##### Cancel a Transaction

```php
$cancel = $this->veritrans->cancel($order_id);
var_dump($cancel);
```




