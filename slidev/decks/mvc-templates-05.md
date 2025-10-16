---
theme: default
title: Twig Templates and Base Layout
info: |
  Introduce Twig, add a shared base layout, and render views from controllers
layout: cover
render_with_liquid: false
highlighter: shiki
lineNumbers: true
drawings:
  persist: false
transition: slide-left
mdc: true
download: true
exportFilename: twig-base-layout
class: text-center
---

# Twig Templates + Base Layout

Template engine • Auto-escape • Reusable layout

---
layout: two-cols
---

# Recap (Project 04)

You built:
- `.env` + `phpdotenv`
- `Database` helper (PDO)
- Contact flow (GET/POST) with prepared statements

::right::

# Today (Project 05)

Add:
- Twig as the view layer
- `templates/` directory
- Shared base layout (`base.html.twig`)
- Render from controllers

---

# Why a Template Engine?

Benefits:
- Security: auto-escaping output
- Separation of concerns: controllers pass data; templates render HTML
- Reuse: base layouts, blocks, partials
- Clarity: fewer `echo`/`htmlspecialchars` in PHP

---

# Install Twig

Commands:
```bash
composer require twig/twig:^3.0
composer dump-autoload
```

Folder:
```
templates/
  base.html.twig
  home.html.twig
```

---

# Bootstrap Twig (Factory)

```php
use Twig\Environment;
use Twig\Loader\FilesystemLoader;
use Twig\TwigFunction;

class TwigFactory {
  private static ?Environment $env = null;
  public static function env(): Environment {
    if (self::$env) return self::$env;
    $loader = new FilesystemLoader(__DIR__.'/../../templates');
    $twig = new Environment($loader, ['cache'=>false, 'autoescape'=>'html']);
    $twig->addFunction(new TwigFunction('path', fn(string $name, array $p=[]): string => $name==='home' ? '/' : '/'.ltrim($name,'/')));
    return self::$env = $twig;
  }
}
```

Notes:
- Auto-escape enabled
- Minimal `path()` helper for links

---

# Base Layout (Overview)

Key blocks in `base.html.twig`:
- `title` – page title
- `meta` – SEO meta tags
- `styles` / `scripts` – page-specific assets
- `flash` – notification messages
- `page_header` – heading
- `content` – main body

Uses Bulma for quick styling.

---

# Extend the Base (Home Page)

{% raw %}
```twig
{% extends 'base.html.twig' %}

{% block title %}{{ pageTitle|default('Home') }} - {{ siteName|default('Site') }}{% endblock %}

{% block page_header %}
  <h1 class="title">Welcome to {{ siteName|default('Site')|e }}</h1>
{% endblock %}

{% block content %}
  <p>This page is rendered with Twig.</p>
{% endblock %}
```
{% endraw %}

---

# Render from a Controller

```php
class HomeController {
  public function index(): void {
    $twig = TwigFactory::env();
    echo $twig->render('home.html.twig', [
      'siteName' => 'My App',
      'pageTitle' => 'Home',
    ]);
  }
}
```

Route:
- GET `/` → `HomeController@index`

---

# Variables, Filters, Control Flow

Examples:
{% raw %}
```twig
{{ name }}           {# variable #}
{{ email|e }}        {# escape #}
{{ total|number_format(2) }}
{% if items|length %} ... {% endif %}
{% for item in items %} ... {% endfor %}
```

Default values:
```twig
{{ pageTitle|default('Page') }}
```
{% endraw %}

---

# Links with `path()`

We added a minimal `path()` function:

{% raw %}
```twig
<a href="{{ path('home') }}">Home</a>
<a href="{{ path('contact') }}">Contact</a>
```
{% endraw %}

Note:
- In larger frameworks, `path()` maps route names to URLs
- Our stub maps `'home' → '/'` and others to `'/name'`

---

# Demo Plan

1) Install Twig
2) Add `templates/` + copy `base.html.twig`
3) Add `TwigFactory` and `path()` helper
4) Create `home.html.twig` extending base
5) Render from controller; test `/`

---

# Common Pitfalls
- Forgetting to enable auto-escape
- Mixing PHP echo in Twig templates
- Not passing required variables from the controller
- Missing `templates/` path or wrong relative path

---

# Q&A / Next Steps

- Partials (`include`), macros, and components
- Layout variants (admin vs public)
- Rendering forms and validation messages
