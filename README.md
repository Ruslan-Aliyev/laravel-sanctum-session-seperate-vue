# Laravel - Sanctum (Session based) - Seperate Vue

Flow:
- Client to server: CRSF cookie request 
    - `XSRF-TOKEN` is obtained from server and saved in the cookies
    - `laravel_session` is obtained from server and saved in the cookies. This cookie holds the user's session
- Client to server: `POST /login` 
- Client to server: `GET /protected-stuff` (protected routes can be accessed because user can be authenticated for the duration of this session)

Configs:
- `config/cors.php`
    - `paths`: lists which Laravel paths can ACCEPT cross-origin requests. 
        - This should be set to `['api/*', 'sanctum/csrf-cookie', 'login', 'register', 'logout']`
    - `allowed_origins`: lists what origins are allowed to SEND requests to Laravel
        - This should be set to `[env('FRONTEND_URL')]`. This is the url of the frontend SPA, it must be in the same domain as your backend API. 
        - Eg: `https://front.website.com` or `http://localhost:5173`
        -  Do NOT put slash at the end!
    - `supports_credentials`: This tells Laravel whether to share cookies with the SPA. A value of `true` will make the `Access-Control-Allow-Credentials` header equal true
        - This corresponds to `resources/js/bootstrap.js`'s `axios.defaults.withCredentials = true`
        - `withCredentials` should be set to `true`, because `XMLHttpRequest` responses from a different domain cannot set cookie values for their own domain unless `withCredentials` is set to `true` before making the request.
- `config/session.php`
    - `'domain' => env('SESSION_DOMAIN')`
        - This is the cookie domain. In other words, it's the top level domain.
        - Eg: `website.com` or `localhost` 
        - Note: Do NOT put trailing slash, url scheme nor port.
        - If you don't set this cookie domain, then the `XSRF-TOKEN` & `laravel_session` cookies won't be there in the communications between your SPA and Laravel.
    - `'driver' => env('SESSION_DRIVER', 'file')`
        - This should be `'cookie'`, not `'file'`
- `config/sanctum.php`
    - `'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS',`
        - Because cookies only get sent with requests to the same domain as the cookie, so you need to make this setting. If the domain sending the request isn't a part of this `SANCTUM_STATEFUL_DOMAINS` list, then the request won't be authenticated.
        - Eg: `front.website.com` or `localhost:5173` 
        - Note: Only include hostname and port.

## Setup

```
composer require laravel/breeze
php artisan breeze:install api # This also installs Sanctum
php artisan migrate --seed
```

You can also make your own `/login` route and controller. It only need to check if the user exists in the database.

Ensure the below is present in `app/Http/Kernel.php` (Update: Laravel 11: `bootstrap/app.php`):
```php
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    ...
```

Comparing `\vendor\laravel\sanctum\src\Http\Middleware\EnsureFrontendRequestsAreStateful.php` and `Kernel.php`'s
```php
protected $middlewareGroups = [
        'web' => [
```
You can see they are almost the same.

What `EnsureFrontendRequestsAreStateful` do is: 
- overrides the session config. It sets http_only to true, meaning that a client-side script has no access to the token. It also sets same_site to “lax”. this will prevent cookies being sent for cross-site requests except for when the request comes from a link to your site from another site.
- If the request is coming from the frontend, these middlewares will be inserted:
    - `EncryptCookies`: Encrypting a cookie means that even if an attacker can gain access to the cookie, modifying its content will result in the cookie being rejected by the server when it is sent back.
    - `AddQueuedCookiesToResponse`: Handles any cookies that have been queued with the Cookie facade.
    - `StartSession`: Sets up the Laravel session along with its session cookie, which it adds to the response.
    - `VerifyCsrfToken`: Checks that everything’s in order with the CSRF token.
- It essentially does the same thing as the default session authentication.

In `routes/api.php`: Protect user with the sanctum middleware:
```php
Route::middleware('auth:sanctum')->get('/protected-stuff', function (Request $request) {
    return 'protected stuff';
});
```

In Vue:
```js
axios.get('http://localhost:8000/sanctum/csrf-cookie').then(response => {
    axios.post('http://localhost:8000/login', this.formData).then(response => {
        console.log('User signed in!');
        axios.get('http://localhost:8000/api/protected-stuff').then(response => {
            console.log(response.data);
        });
    }).catch(error => console.log(error)); // credentials didn't match
});
```

If you decide to run `npm run dev` and serve the frontend on a different port, you also need to make the configs described above.

![](/Illustrations/results.png)

---

## Tutorials

- https://laravel-news.com/using-sanctum-to-authenticate-a-react-spa
- https://www.youtube.com/watch?v=2zKoS8GsKK8
- https://www.youtube.com/watch?v=gKC7yvllsPE
