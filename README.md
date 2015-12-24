# LaravelPagseguro

### Instruções

Esse pacote utiliza a lib phpsc/pagseguro, gerando um ServiceProvider para aplicações Laravel 5.

### Instalação

Adicione este repositório no seu composer:

```json
"repositories": [
    {
        "type": "git",
        "url": "https://github.com/joaosalless/laravel-pagseguro"
    }
],
```

Rode: composer require joaosalless/laravel-pagseguro:@dev

Adicione o seguinte service provider em seu arquivo config/app.php:

```php
'providers' => [
    //...
    'LaravelPagseguro\LaravelPagseguroServiceProvider'
]
```

Rode o seguinte comando no artisan:

```bash
php artisan vendor:publish
```

Edite o arquivo *config/pagseguro.php*, entrando com o ambiente, email e token de sua conta pagseguro.

## Exemplos de utilização no Laravel 5.1

### Controller para Checkout

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests;
use App\Repositories\UserRepository;
use App\Repositories\OrderRepository;
use PHPSC\PagSeguro\Items\Item;
use PHPSC\PagSeguro\Customer\Phone;
use PHPSC\PagSeguro\Customer\Address;
use PHPSC\PagSeguro\Customer\Customer;
use PHPSC\PagSeguro\Requests\Checkout\CheckoutService;
use PHPSC\PagSeguro\Shipping\Shipping;

class PagseguroController extends Controller
{
    private $userRepository;
    private $orderRepository;
    private $checkoutService;

    public function __construct(
        UserRepository $userRepository,
        OrderRepository $orderRepository,
        CheckoutService $checkoutService
    ) {
        $this->userRepository  = $userRepository;
        $this->orderRepository = $orderRepository;
        $this->checkoutService = $checkoutService;
    }

    public function checkout($orderId)
    {
        try {
            $order = $this->orderRepository->find($orderId);
            $user  = $this->userRepository->with('client')->find(Auth::user()->id);

            $customer = new Customer(
                'emaildocliente@user.com',
                'Nome do Cliente',
                new Phone('11', '999998888'),
                new Address(
                    $state      = 'SP',
                    $city       = 'São Paulo',
                    $postalCode = '01323000',
                    $district   = 'Bairro',
                    $street     = 'Nome da Rua',
                    $number     = '100',
                    $complement = 'AP 101'
                )
                // $order->user->email,
                // $order->user->name,
                // new Phone($order->client->phone_area, $order->client->phone),
                // new Address(
                //     $order->client->state,
                //     $order->client->city,
                //     $order->client->district,
                //     $order->client->zipcode,
                //     $order->client->street,
                //     $order->client->street_number,
                //     null
                // )
            );

            $shipping = new Shipping(3, $customer->getAddress(), null);

            $checkout = $this->checkoutService->createCheckoutBuilder()
                ->setReference(config('pagseguro.reference.prefix', 'ORDER_ID_').$order->id) // Aqui deve ser informado o ID do pedido (orderID)
                ->setCustomer($customer)
                ->setShipping($shipping);

            foreach ($order->items as $item) {
                $checkout->addItem(new Item(
                    $item->product->id,
                    $item->product->name,
                    $item->price,
                    $item->qtd,
                    null,
                    null
                ));
            }

            $checkout->setRedirectTo(route('pagseguro.retorno.checkout'));
            $checkout = $checkout->getCheckout();
            $response = $this->checkoutService->checkout($checkout);

            return redirect($response->getRedirectionUrl());

        } catch (Exception $error) {
            echo $error->getMessage();
        }
    }
}
````

### Controller para Retorno Automático e Notificações

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests;
use Illuminate\Http\Request;
use App\Http\Controllers\Controller;
use App\Repositories\UserRepository;
use App\Repositories\OrderRepository;
use App\Repositories\TransactionRepository;
use App\Http\Requests\PagSeguro\PagseguroRetornoRequest;
use PHPSC\PagSeguro\Purchases\Transactions\Locator as TransactionLocator;
use PHPSC\PagSeguro\Purchases\Subscriptions\Locator as SubscriptionLocator;

class RetornoController extends Controller
{
    private $userRepository;
    private $orderRepository;
    private $transactionRepository;

    public function __construct(
        UserRepository $userRepository,
        OrderRepository $orderRepository,
        TransactionRepository $transactionRepository
    ) {
        $this->userRepository        = $userRepository;
        $this->orderRepository       = $orderRepository;
        $this->transactionRepository = $transactionRepository;
    }

    /**
     * -------------------------------------------------------------------------
     * Redirecionamento com o código da transação após o checkout
     * -------------------------------------------------------------------------
     * Recebe a chamada do tipo GET redirecionamento do PagSeguro após
     * o pagamento com o código da transação.
     *
     * Exemplo de Rota:
     * Route::get('pagseguro/retorno/checkout', ['as' => 'pagseguro.retorno.checkout', 'uses' => 'Api\PagSeguro\RetornoController@retornoCheckout']);
     *
     * Observação:
     * Na função que executa o checkout deve ser setada a url de retorno assim
     * $checkout->setRedirectTo(route('pagseguro.retorno.checkout'));
     **/
    public function retornoCheckout(Request $request, TransactionLocator $transactionLocator)
    {
        try {
            // Solicita a transação os dados da transação ao PagSeguro
            $transactionPagseguro = $transactionLocator->getByCode($request->get('transaction_id'));

            // Pega o ID do pedido que gerou a transação
            $orderId = str_replace(config('pagseguro.reference.prefix', 'ORDER_ID_'), '', $transactionPagseguro->getDetails()->getReference());

            // $order = $this->orderRepository->find($orderId);
            //
            // $transaction = $this->transactionRepository->firstOrNew([
            //     'transaction_code' => $transactionPagseguro->getDetails()->getCode(),
            //     'order_id' => $orderId
            // ]);
            //
            // $transaction->gateway = 'PagSeguro';
            // $transaction->payment_method_type_id = $transactionPagseguro->getPayment()->getPaymentMethod()->getType();
            // $transaction->payment_method_code_id = $transactionPagseguro->getPayment()->getPaymentMethod()->getCode();
            // $transaction->amount = $transactionPagseguro->getPayment()->getGrossAmount();
            // $transaction->status = $transactionPagseguro->getDetails()->getStatus();
            // $transaction->save();

            dd($transactionPagseguro);

        } catch (Exception $e) {
            dd($e->getMessage()); // Exibe na tela a mensagem de erro caso exista erro.
        }
    }

    /**
     * -------------------------------------------------------------------------
     * Retorno automático para Assinaturas
     * -------------------------------------------------------------------------
     * Recebe a chamada do tipo GET redirecionamento do PagSeguro após
     * o pagamento com o código da transação.
     *
     * Exemplo de Rota:
     * Route::get('pagseguro/retorno/subscription', ['as' => 'pagseguro.retorno.subscription', 'uses' => 'Api\PagSeguro\RetornoController@retornoSubscription']);
     *
     * Observação:
     * Na função que executa o checkout da assinatura deve ser setada a url de retorno assim
     * $checkout->setRedirectTo(route('pagseguro.retorno.subscription'));
     **/
    public function retornoSubscription(Request $request, SubscriptionLocator $subscriptionLocator)
    {
        try {
            $service = $subscriptionLocator;

            // Solicita a transação os dados da assinatura ao PagSeguro
            $subscription = $service->getByCode($request->get('transaction_id'));

            dd($subscription);

        } catch (Exception $e) {
            dd($e->getMessage()); // Exibe na tela a mensagem de erro caso exista erro.
        }
    }

    /**
     * -------------------------------------------------------------------------
     * Notificação de transações
     * -------------------------------------------------------------------------
     * Recebe uma chamada do tipo POST enviada pelo PagSeguro contendo os campos
     * "notificationCode" = XXXXXX-XXXXXXXXXXXX-XXXXXXXXXXXX-XXXXXX
     * "notificationType" = transaction | preApproval
     *
     * Exemplo de Rota:
     * Route::Post('pagseguro/retorno/notification', ['as' => 'pagseguro.retorno.notification', 'uses' => 'Api\PagSeguro\RetornoController@retornoNotification']);
     *
     * Observação:
     * Adicionar a url desta rota na lista de excessões do meddleware VerifyCsrfToken;
     **/
    public function retornoNotification(
        Request $request,
        TransactionLocator $transactionLocator,
        SubscriptionLocator $subscriptionLocator
    ) {
        try {
            // Cria instância do serviço de acordo com o tipo da notificação
            $service = $request->get('notificationType') == 'preApproval'
                       ? $subscriptionLocator
                       : $transactionLocator;

            // Solicita os dados da transação ou assinatura atualizados ao PagSeguro
            $purchase = $service->getByNotification($request->get('notificationCode'));

            dd($purchase);

        } catch (Exception $e) {
            dd($e->getMessage()); // Exibe na tela a mensagem de erro caso exista erro.
        }
    }
}

````

### Rotas para Retorno e Notificações

```php
<?php
/** ------------------------------------------------------------------------
 *  PagSeguro / Retorno
 *  ------------------------------------------------------------------------
 */
Route::group(['prefix' => 'pagseguro', 'as' => 'pagseguro.', 'middleware' => 'cors'], function () {
    Route::get('retorno/checkout',      ['as'   => 'retorno.checkout',     'uses' => 'PagSeguroRetornoController@retornoCheckout']);
    Route::get('retorno/subscription',  ['as'   => 'retorno.subscription', 'uses' => 'PagSeguroRetornoController@retornoSubscription']);
    Route::post('retorno/notification', ['as'   => 'retorno.notification', 'uses' => 'PagSeguroRetornoController@retornoNotification']);
});
```
