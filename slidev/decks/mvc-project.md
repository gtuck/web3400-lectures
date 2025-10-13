---
theme: default
title: MVC+ Router, Namespaces & Autoloading
info: |
  Scaling your MVC refactor with a front controller, simple routing, and PSR-4 autoloading
layout: cover
highlighter: shiki
lineNumbers: true
drawings:
  persist: false
transition: slide-left
mdc: true
download: true
exportFilename: mvc-plus-router-slides
class: text-center
---

# MVC+ Router, Namespaces & Autoloading

From a basic MVC refactor → a small structured MVC app

---
layout: two-cols
---

# Recap (Project 02)

What you built:
- `Model` with DB logic (PDO)
- `View` with HTML + escaping
- `Controller` coordinating model → view
- Clean `index.php` entry point

::right::

# Next Step (Project 03)

Add:
- Front controller (`public/index.php`)
- Router + routes file
- Namespaces + PSR-4 autoloading
- Base `Controller` with `render()` helper

---

# Target Structure

```
composer.json
public/
  index.php
src/
  Controller.php
  Router.php
  Controllers/
    HomeController.php
  Models/
    Blog.php
  Routes/
    index.php
  Views/
    index.php
```

---

# Composer (What & Why)

What it is:
- PHP’s dependency manager. Installs packages (from Packagist), manages versions, and generates autoloaders.

Key files/dirs:
- `composer.json` – project manifest (deps, autoload rules)
- `composer.lock` – exact versions installed
- `vendor/` – installed packages + `vendor/autoload.php`

Common commands:
- `composer init` / `composer require vendor/package`
- `composer install` / `composer update`
- `composer dump-autoload` (regenerates autoload map)

Why we use it here:
- Enables PSR‑4 autoloading so classes load automatically by namespace
- Simplifies project structure (no manual require chains)
- Standard approach in modern PHP apps

---

# PSR-4 (PHP Standard Recommendation 4) Explained

What it is:
- A standard for autoloading classes based on namespaces and directory layout.

Core idea:
- Map a namespace prefix to a base directory.
- Each namespace segment → a subdirectory.
- The class name → the filename with `.php`.

Example mapping:
```json
{
  "autoload": {
    "psr-4": { "App\\": "src/" }
  }
}
```
---

# PSR-4 Cont.

Resolution examples:
- `App\\Controllers\\HomeController` → `src/Controllers/HomeController.php`
- `App\\Models\\Blog` → `src/Models/Blog.php`

Namespace declarations must match paths:
```php
// src/Controllers/HomeController.php
namespace App\Controllers;
class HomeController {}
```

Caveats:
- Case-sensitive on Unix-like systems.
- After changes, run `composer dump-autoload`.
- Include `vendor/autoload.php` once; no manual `require` chains.

---

# Composer Autoloading (PSR-4)

`composer.json`
```json
{
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  }
}
```

Run: `composer dump-autoload`

Benefits:
- No more `require` chains
- Namespaces map to folders

---

# Namespaces

Examples:
```php
namespace App;                 // src/Controller.php
namespace App\Controllers;     // src/Controllers/HomeController.php
namespace App\Models;          // src/Models/Blog.php
```

Use statements:
```php
use App\Models\Blog;
```

Conventions:
- Classes: StudlyCaps
- Methods: camelCase

---

# Front Controller (public/index.php)

Single entry point:
```php
<?php
require '../vendor/autoload.php';

$router = require '../src/Routes/index.php';
```

Centralizes bootstrapping:
- Autoload
- Routing
- Error handling (extend later)

---

# Router & Dispatch (Concepts)

What a Router does:
- Keeps a route table keyed by HTTP method + path
- Matches the current request (method + URI path, without the query string)
- Maps to a controller class and action method

What Dispatch means:
- Instantiate the matched controller
- Invoke the mapped action method
- Optionally pass path params/body data (simple version: none)
- Return a response (usually by rendering a view)

---

# Router & Dispatch Cont.

Request flow:
```
Request → Router.match(method, path)
       → Controller@action
       → Model (fetch data)
       → View (render output)
       → Response
```

Good practices:
- Register routes separately from dispatch
- Ignore the query string when matching paths
- Return clear errors for unknown routes (404/No route) or methods (405)
- Keep controllers thin; no routing logic inside controllers

---

# Router (src/Router.php)

Responsibilities:
- Register routes for GET/POST
- Dispatch based on URI + HTTP method

Key parts:
```php
class Router {
  protected $routes = [];

  private function addRoute($route, $controller, $action, $method) {
    $this->routes[$method][$route] = ['controller' => $controller, 'action' => $action];
  }

  public function get($route, $controller, $action)  { $this->addRoute($route, $controller, $action, 'GET'); }
  public function post($route, $controller, $action) { $this->addRoute($route, $controller, $action, 'POST'); }

  public function dispatch() {
    $uri = strtok($_SERVER['REQUEST_URI'], '?');
    $method = $_SERVER['REQUEST_METHOD'];
    if (!isset($this->routes[$method][$uri])) throw new \Exception("No route for $method $uri");
    $controller = new ($this->routes[$method][$uri]['controller']);
    $action = $this->routes[$method][$uri]['action'];
    $controller->$action();
  }
}
```

---

# Routes (src/Routes/index.php)

```php
use App\Controllers\HomeController;
use App\Router;

$router = new Router();
$router->get('/', HomeController::class, 'index');
$router->dispatch();
```

- Clear mapping: path → controller@action
- Add more routes as features grow

---

# Base Controller (src/Controller.php)

`render()` helper keeps controllers thin:
```php
class Controller {
  protected function render($view, $data = []) {
    extract($data);
    include "Views/$view.php";
  }
}
```

Use it:
```php
$this->render('index', ['posts' => $posts]);
```

---

# Model (src/Models/Blog.php)

Move PDO code from P02 into a namespaced model:
```php
class Blog {
  public function getPosts(): array {
    $dsn = 'mysql:host=db;dbname=web3400;charset=UTF8';
    $pdo = new \PDO($dsn, 'web3400', 'password', [
      \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
      \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
    ]);
    return $pdo->query('SELECT * FROM posts')->fetchAll();
  }
}
```

---

# Controller (src/Controllers/HomeController.php)

```php
use App\Models\Blog;

class HomeController extends Controller {
  public function index() {
    $posts = (new Blog())->getPosts();
    $this->render('index', ['posts' => $posts]);
  }
}
```

---

# View (src/Views/index.php)

```php
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>Blog Posts</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
</head>
<body>
  <h1>Blog Posts</h1>
  <?php foreach ($posts as $post): ?>
    <h2><?= htmlspecialchars($post['title']) ?></h2>
    <p><?= htmlspecialchars($post['body']) ?></p>
  <?php endforeach; ?>
</body>
</html>
```

---

# Request Flow

```
User → public/index.php → Router → Controller → Model → Controller → View → Response
```

- Single entry point
- Clear responsibilities per layer

---

# Run

From project root:
```
composer dump-autoload
php -S 0.0.0.0:8000 -t public
```

Open: http://localhost:8000/

---

# Common Pitfalls
- Missing `composer dump-autoload`
- Namespace doesn’t match folder path
- Not escaping output in views
- Router URI includes query string (strip it)

---

# Your Turn
- Bring your P02 code
- Restructure into the P03 layout
- Keep behavior the same, improve architecture
---
