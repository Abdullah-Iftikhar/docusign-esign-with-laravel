# Integrate Docusign API esign in LARAVEL

### Installing Dependencies

To install the required PHP dependencies, run the following Composer command in your project directory

```bash
composer require docusign/esign-client
```

Get all these values from your *DocuSign* account.

### Set Your .env

```dotenv
DOCUSIGN_USER_ID = *********
DOCUSIGN_API_ACCOUNT_ID = *********
DOCUSIGN_ACCOUNT_BASE_URI = *********
DOCUSIGN_APP_NAME = *********
DOCUSIGN_INTEGRATION_KEY = *********
DOCUSIGN_SECRET_KEY = *********
DOCUSIGN_ACCOUNT_BASE_URI_API = *********
DOCUSIGN_SCOPE = *********
```

Copy this code for *web.php*

### Routes

```php
Route::get('docusign', [DocusignController::class, 'index'])->name('docusign');
Route::get('connect-docusign', [DocusignController::class, 'connectDocusign'])->name('connect.docusign');
Route::get('docusign/callback', [DocusignController::class, 'callback'])->name('docusign.callback');
Route::get('sign-document', [DocusignController::class, 'signDocument'])->name('docusign.sign');
```

Just get the authenticated token through API and set this in the session for *e-signature* requests.

### Login Through API & Get The Authenticated Token

```php
//DocuSign Auth Token
$apiClient = new ApiClient();
$apiClient->getOAuth()->setOAuthBasePath($this->DOCUSIGN_ACCOUNT_BASE_URI);
$docuSignAuthCode = $this->getToken($apiClient);
session()->put('docusign_auth_code', $docuSignAuthCode);
//End Docusign Auth Token

private function getToken(ApiClient $apiClient) : string{
    try {
        $privateKey = file_get_contents(public_path('private.key'),true); // This is your application key creat on DocuSign
        $response = $apiClient->requestJWTUserToken(
            $ikey = $this->DOCUSIGN_INTEGRATION_KEY,
            $userId =  $this->DOCUSIGN_USER_ID,
            $key = $privateKey,
            $scope = $this->DOCUSIGN_SCOPE
        );
        $token = $response[0];
        $accessToken = $token->getAccessToken();
    } catch (\Exception $th) {
        throw $th;
    }
    return $accessToken;
}
```

You can set the signature *tabs* on specific places and *initials*.

### DocusignController.php

```php
<?php

namespace App\Http\Controllers\InternalAgent;

use App\Http\Controllers\Controller;
use DocuSign\eSign\Api\EnvelopesApi;
use DocuSign\eSign\Client\ApiClient;
use DocuSign\eSign\Configuration;
use DocuSign\eSign\Model\InitialHere;
use Exception;
use Session;

class DocusignController extends Controller
{

    /** hold config value */
    private $config;

    private $signer_client_id = 1000; # Used to indicate that the signer will use embedded

    /** Specific template arguments */
    private $args;

    private $clientId;
    private $clinetSceret;
    private $URL;
    private $accountId;
    private $baseUrl;
    private $dsReturnUrl;

    public function __construct()
    {
        $this->clientId = env('DOCUSIGN_INTEGRATION_KEY');
        $this->clinetSceret = env('DOCUSIGN_SECRET_KEY');
        $this->accountId = env('DOCUSIGN_API_ACCOUNT_ID');
        $this->baseUrl = "https://".env('DOCUSIGN_ACCOUNT_BASE_URI_API');
    }

    public function signDocument()
    {
        try {
            $this->dsReturnUrl = route('home');
            $this->args = $this->getTemplateArgs();
            $args = $this->args;
            $envelope_args = $args["envelope_args"];

            # Create the envelope request object
            $envelope_definition = $this->make_envelope($args["envelope_args"]);
            $envelope_api = $this->getEnvelopeApi();
            # Call Envelopes::create API method
            # Exceptions will be caught by the calling function

            $api_client = new \DocuSign\eSign\client\ApiClient($this->config);

            $envelope_api = new \DocuSign\eSign\Api\EnvelopesApi($api_client);

            $results = $envelope_api->createEnvelope('cf56cad6-f8ce-4274-ab9b-8c7e07c0511a', $envelope_definition);

            $envelope_id = $results->getEnvelopeId();

            session()->put('envelope_id', $envelope_id);

            $authentication_method = 'None'; # How is this application authenticating
            # the signer? See the `authenticationMethod' definition
            # https://developers.docusign.com/esign-rest-api/reference/Envelopes/EnvelopeViews/createRecipient
            $recipient_view_request = new \DocuSign\eSign\Model\RecipientViewRequest([
                'authentication_method' => $authentication_method,
                'client_user_id' => $envelope_args['signer_client_id'],
                'recipient_id' => auth()->user()->id,
                'return_url' => $envelope_args['ds_return_url'],
                'user_name' => auth()->user()->first_name . ' ' . auth()->user()->last_name,
                'email' => auth()->user()->email
            ]);

            $results = $envelope_api->createRecipientView($args['account_id'], $envelope_id, $recipient_view_request);

            return redirect()->to($results['url']);
        } catch (Exception $e) {
            return response()->json([
                'success' => false,
                'message' => $e->getMessage(),
            ], 404);
        }
    }

    private function make_envelope($args)
    {
        $demo_docs_path = YOUR_FILE_PATH;

        $arrContextOptions = array(
            "ssl" => array(
                "verify_peer" => false,
                "verify_peer_name" => false,
            ),
        );

        $content_bytes = file_get_contents($demo_docs_path, false, stream_context_create($arrContextOptions));
     
        $base64_file_content = base64_encode($content_bytes);
      
        # Create the document model
        $documentId = rand(1, 10) . auth()->user()->id . rand(1, 10);
        $document = new \DocuSign\eSign\Model\Document([ # create the DocuSign document object
            'document_base64' => $base64_file_content,
            'name' => 'Example document', # can be different from the actual file name
            'file_extension' => 'pdf', # many different document types are accepted
            'document_id' => $documentId, # a label used to reference the doc
        ]);
        session()->put('document_id', $documentId);

        # Create the signer recipient model
        $signer = new \DocuSign\eSign\Model\Signer([ # The signer
            'email' => 'info@example.com',
            'name' => 'test name',
            'recipient_id' => auth()->user()->id,
            'routing_order' => "1",
            # Setting the client_user_id marks the signer as embedded
            'client_user_id' => $args['signer_client_id'],
        ]);
        # Create a sign_here tab (field on the document)


        $accompanyingSignature = new \DocuSign\eSign\Model\SignHere([
            'anchor_string' => 'unique_key_1',
//This key also available in your docusment. Docusign will identify this text key and place the signature on this key place automatically
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '-10',
            'anchor_x_offset' => '270',
            'optional' => false,
        ]);

        $signatureAuthorization = new \DocuSign\eSign\Model\SignHere([
            'anchor_string' => 'unique_key_2', // Customize this anchor string
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '-10', // Customize the Y offset
            'anchor_x_offset' => '270', // Customize the X offset
            'optional' => false,
        ]);


        $agencyAuthorization = new \DocuSign\eSign\Model\SignHere([
            'anchor_string' => 'unique_key_3', // Customize this anchor string
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '-7', // Customize the Y offset
            'anchor_x_offset' => '210', // Customize the X offset
            'optional' => false,
        ]);

        $agAintial = new InitialHere([
            'anchor_string' => 'intial_unique_key_1', // Customize this anchor string
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '7', // Customize the Y offset
            'anchor_x_offset' => '35', // Customize the X offset
            'optional' => false,
            'font_size' => 'Size12',
        ]);

        $agBintial = new InitialHere([
            'anchor_string' => 'intial_unique_key_2', // Customize this anchor string
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '7', // Customize the Y offset
            'anchor_x_offset' => '35', // Customize the X offset
            'optional' => false,
            'font_size' => 'Size12',
        ]);

        $agCintial = new InitialHere([
            'anchor_string' => 'intial_unique_key_3', // Customize this anchor string
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '7', // Customize the Y offset
            'anchor_x_offset' => '35', // Customize the X offset
            'optional' => false,
            'font_size' => 'Size12',
        ]);

        $agDintial = new InitialHere([
            'anchor_string' => 'intial_unique_key_4', // Customize this anchor string
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '7', // Customize the Y offset
            'anchor_x_offset' => '35', // Customize the X offset
            'optional' => false,
            'font_size' => 'Size12',
        ]);

        $agEintial = new InitialHere([
            'anchor_string' => 'intial_unique_key_5', // Customize this anchor string
            'anchor_units' => 'pixels',
            'anchor_y_offset' => '7', // Customize the Y offset
            'anchor_x_offset' => '35', // Customize the X offset
            'optional' => false,
            'font_size' => 'Size12',
        ]);

        $signatures = [
            $accompanyingSignature,
            $signatureAuthorization,
            $agencyAuthorization,
        ];

        $intialSignature = [
            $agAintial,
            $agBintial,
            $agCintial,
            $agDintial,
            $agEintial,
        ];

        # Add the tabs model (including the sign_here tab) to the signer
        # The Tabs object wants arrays of the different field/tab types

        $signer->settabs(new \DocuSign\eSign\Model\Tabs(
            [
                'sign_here_tabs' => $signatures,
                'initial_here_tabs' => $intialSignature,
            ]
        ));
        # Next, create the top level envelope definition and populate it.

        $envelope_definition = new \DocuSign\eSign\Model\EnvelopeDefinition([
            'email_subject' => "Test Subject",
            'documents' => [$document],
            # The Recipients object wants arrays for each recipient type
            'recipients' => new \DocuSign\eSign\Model\Recipients(['signers' => [$signer]]),
            'status' => "sent", # requests that the envelope be created and sent.
        ]);

        return $envelope_definition;
    }

    /**
     * Getter for the EnvelopesApi
     */
    public function getEnvelopeApi(): EnvelopesApi
    {
        $this->config = new Configuration();
        $this->config->setHost($this->args['base_path']);
        $this->config->addDefaultHeader('Authorization', 'Bearer ' . $this->args['ds_access_token']);
        $this->apiClient = new ApiClient($this->config);
        return new EnvelopesApi($this->apiClient);
    }

    /**
     * Get specific template arguments
     *
     * @return array
     */
    private function getTemplateArgs()
    {
        $envelope_args = [
            'signer_client_id' => auth()->user()->id,
            'ds_return_url' => $this->dsReturnUrl,
        ];
        $args = [
            'account_id' => $this->accountId,
            'base_path' => $this->baseUrl,
            'ds_access_token' => Session::get('docusign_auth_code'),
            'envelope_args' => $envelope_args
        ];
        return $args;
    }
}
```


