# Integrate Docusign API esign in LARAVEL

### Installing Dependencies

To install the required PHP dependencies, run the following Composer command in your project directory:

```bash
composer require docusign/esign-client
```
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

### Routes

```php
Route::get('docusign', [DocusignController::class, 'index'])->name('internal.agent.docusign');
Route::get('connect-docusign', [DocusignController::class, 'connectDocusign'])->name('internal.agent.connect.docusign');
Route::get('docusign/callback', [DocusignController::class, 'callback'])->name('internal.agent.docusign.callback');
Route::get('sign-document/{key?}', [DocusignController::class, 'signDocument'])->name('internal.agent.docusign.sign');
```

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
        $privateKey = file_get_contents(public_path('private.key'),true);
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
