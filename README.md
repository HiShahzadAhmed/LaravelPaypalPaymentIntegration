# Laravel Paypal Payment Integration

Laravel Paypal payment integration in Laravel 6/7/8

## Usage



### Install the Rest API SDK Package in your Project

Add this command to composer.json file and update composer in CMD

```bash
"paypal/rest-api-sdk-php": "^1.14"
```



### Copy the paypal.php class

Copy the paypal.php file from repository and add into config directory of project


### Set Credentials in env file

Add this command to composer.json file and update composer in CMD
Go to https://www.sandbox.paypal.com/us/signin to get the testing credentials

```bash
PAYPAL_CLIENT_ID=
PAYPAL_SECRET_ID=
PAYPAL_MODE=sandbox
```



### Creating a Trait Class

Create a folder in App direcotry with Traits (e.g. App/Traits).
Create a new Class file with PayPalPayment.php
Copy the follwing code and paste in the file

```php

<?php

namespace App\Traits;
use Session;
use Redirect;
use PayPal\Api\Amount;
use PayPal\Api\Details;
use PayPal\Api\Item;
use PayPal\Api\ItemList;
use PayPal\Api\Payer;
use PayPal\Api\Payment;
use PayPal\Api\RedirectUrls;
use PayPal\Api\ExecutePayment;
use PayPal\Api\PaymentExecution;
use PayPal\Api\Transaction;
use PayPal\Rest\ApiContext;
use PayPal\Auth\OAuthTokenCredential;




trait PaypalPayment{
   
    private $_api_context;

    public function __construct()
    {


        $paypal_configuration = \Config::get('paypal');
        $this->_api_context = new ApiContext(new OAuthTokenCredential($paypal_configuration['client_id'], $paypal_configuration['secret']));
        $this->_api_context->setConfig($paypal_configuration['settings']);
    
    }




    public function makePaypalCheckout($data)
    {


            $price       =  $data['price'];
            $email       =  $data['payer_email'];
            $success_url = $data['success_url'];
            $cancel_url  = $data['cancel_url'];
            $currency    = $data['currency'];
            $quantity    = $data['quantity'];
            $description = $data['description'];


            $payer = new Payer();
            $payer->setPaymentMethod('paypal');

            $item_1 = new Item();


            $item_1->setName($email)
                ->setCurrency($currency)
                ->setQuantity($quantity)
                ->setPrice($price);


            $item_list = new ItemList();
            $item_list->setItems(array($item_1));

            $amount = new Amount();
            $amount->setCurrency($currency)
                ->setTotal($price);

            $transaction = new Transaction();
            $transaction->setAmount($amount)
                ->setItemList($item_list)
                ->setDescription($description);


            $redirect_urls = new RedirectUrls();
            $redirect_urls->setReturnUrl($success_url)->setCancelUrl($cancel_url);


            $payment = new Payment();
            $payment->setIntent('Sale')
                ->setPayer($payer)
                ->setRedirectUrls($redirect_urls)
                ->setTransactions(array($transaction));
            try 
            {
                $payment->create($this->_api_context);
            } 
            catch (\PayPal\Exception\PPConnectionException $ex) 
            {
                if (\Config::get('app.debug')) {
                    return back()->with('error','Connection timeout');

                } else {

                    return back()->with('error','Some error occur, sorry for inconvenient');
                }
            }


               foreach($payment->getLinks() as $link) {
                    if($link->getRel() == 'approval_url') {
                        $redirect_url = $link->getHref();
                        break;
                    }
                }
                Session::put('paypal_payment_id', $payment->getId());

                if(isset($redirect_url)) 
                {
                    return Redirect::away($redirect_url);
                }
                return back()->with('error','Unknown error occurred');


    }



    public function paypalPaymentSuceess($request)
    {

        $payment_id = Session::get('paypal_payment_id');

        if ($request['PayerID'] == "" || $request['token'] == "" )  
        {
            return redirect()->route('index')->with('error','Payment Failed. Try again later.');
        }
        
        $payment = Payment::get($payment_id, $this->_api_context);
        $execution = new PaymentExecution();
        $execution->setPayerId($request['PayerID']);

        Session::forget('paypal_payment_id');

        return $result = $payment->execute($execution, $this->_api_context);
    }




}



```



### Preparing your Routes & Controller for searching

You'll need to create methods to link from web.php file for searching. Make a route e.g.

```bash

Route::get('/make/paypal/checkout', 'PaymentController@makePaypalCheckout')->name('make.paypal.checkout');

Route::get('payment/paypal/success', 'PaymentController@paymentPaypalSuccess')->name('payment.paypal.success');

Route::get('payment/paypal/cancel', 'PaymentController@paymentPaypalCancel')->name('payment.paypal.cancel');
```

Your Controller will be something like this with methods

```php

<?php
  
namespace App\Http\Controllers;
  
use Illuminate\Http\Request;
use App\Traits\SearchModel; //ADD NEWLY CREATED TRAIT NAMESPACE


class PaymentController extends Controller
{
    use PaypalPayment; //CALL THE TRAIT IN CONTROLLER
    
    public function makePaypalCheckout()
    {
        $data = [
                'price'             => 'AMOUNT_TO_CHARGE',
                'payer_email'       => 'PAYER_EMAIL_ADDRESS',
                'currency'          => 'USD',
                'quantity'          => 1,
                'description'       => 'DESCRIPTION_OF_PAYEMNT',
                'success_url'       => route('payment.paypal.success'), // PASS THE SUCCESS RESPONSE ROUTE NAME
                'cancel_url'        => route('payment.paypal.cancel')  // PASS THE CANCEL RESPONSE ROUTE NAME
            ];
            
            return $this->makePaypalCheckout($data);        
    }
    
    public function paypalPaymentSuccess(Request $request)
    {   

         $response = $this->paypalPaymentSuceess($request->all());

        if ($response->getState() == 'approved') 
        {
            //DO YOUR TRANSACTION LOGIC HERE TO STORE RECORD IN DATABASE
        }

        return redirect()->route('index')->with('error','Payment failed! Try again later.');
    }
    
    public function paymentPaypalCancel()
    {
        return 'You've cancelled the transaction';
    }

    
    
    

}
```
