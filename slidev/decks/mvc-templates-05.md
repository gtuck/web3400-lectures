---
theme: default
title: Vanilla PHP Template System (Layouts, Sections, Partials) + Flash + PRG
info: |
  Build a tiny templating engine, integrate with MVC, add flash notifications and POST/Redirect/GET
layout: cover
highlighter: shiki
lineNumbers: true
drawings:
  persist: false
transition: slide-left
mdc: true
download: true
exportFilename: vanilla-php-templates-flash-prg
class: text-center
---

# Vanilla PHP Template System

Layouts • Sections • Partials • Escaping • Flash • PRG

---
layout: two-cols
---

# Recap (Project 04)

You built:
- Dotenv config + `.env/.env.example`
- `Database::pdo()` (centralized PDO)
- `BaseModel` + generator
- Contact form (GET/POST)

::right::

# Today (Project 05)

Add:
- Tiny `View` class (output buffering)
- Layouts, named sections, partials
- Shared data + escape helper
- Flash notifications (session)
- PRG (303 redirect after POST)

---

# Why a Template System?

Problems it solves:
- Reuse shared layout chrome (head/nav/footer)
- Keep views simple and structured
- Encapsulate escaping/partials helpers

Constraints:
- Vanilla PHP (no external template libs)
- Minimal API; easy to reason about

---

# `View` API (at a glance)

Helpers available inside templates via `$this`:
- `layout('layouts/main')`
- `start('content')` ... `end()`
- `section('content')`
- `insert('partials/nav', ['key'=> 'val'])`
- `share([...])` (controller side)
- `e($value)` (escape)

---

# Engine (Core Idea)

Output buffering + include files:
```php
ob_start();
extract($vars, EXTR_SKIP);
include $path;
$content = ob_get_clean();
```

- Store named sections during `start()/end()`
- Render view first; then layout if declared

---

# Layout + Sections (Example)

Layout (`layouts/main.php`):
```php
<html>
  <head><?php $this->insert('partials/head', ['title' => $title ?? 'Home']) ?></head>
  <body>
    <?php $this->insert('partials/nav') ?>
    <?php $this->insert('partials/flash') ?>
    <main><?php $this->section('content') ?></main>
    <?php $this->insert('partials/footer') ?>
  </body>
</html>
```

View (`index.php`):
```php
<?php $this->layout('layouts/main') ?>
<?php $this->start('content') ?>
  <h1><?= $this->e($siteName) ?></h1>
<?php $this->end() ?>
```

---

# Partials + Shared Data

Shared in base controller:
```php
$this->view->share(['siteName' => $_ENV['SITE_NAME'] ?? 'My PHP Site']);
```

Partial (`partials/nav.php`):
```php
<a href="/"><?= $this->e($siteName ?? 'Site') ?></a>
```

---

# Escape Helper

Use `$this->e()` for untrusted values:
```php
<h2><?= $this->e($post['title']) ?></h2>
<p><?= $this->e($post['body']) ?></p>
```

Avoid echoing raw request/DB values in templates.

---

# Controller Integration

Base controller:
- Creates `View` with `__DIR__ . '/Views'`
- Shares `SITE_NAME`, `SITE_EMAIL`, `SITE_PHONE` from env
- Adds `flash($text, $type)` helper
- Adds `redirect($path, $status=303)` helper

---

# Flash Notifications

Session-backed messages:
```php
$this->flash('Saved!', 'is-success');
```

Partial renders + clears:
```php
<?php if (!empty($_SESSION['messages'])): ?>
  <?php foreach ($_SESSION['messages'] as $m): ?>
    <div class="notification <?= $this->e($m['type']) ?>"><?= $this->e($m['text']) ?></div>
  <?php endforeach; $_SESSION['messages'] = []; ?>
<?php endif; ?>
```

Remember to `session_start()` in `public/index.php`.

---

# PRG (POST/Redirect/GET)

Flow:
1) POST /contact → validate + persist
2) Flash success
3) Redirect (303) to GET /contact
4) GET renders with flash (no re-submit on refresh)

Benefits:
- Prevents duplicate submissions
- Clear navigation behavior

---

# Demo Plan

1) Add `View` class
2) Wire base controller + share env vars
3) Build `layouts/main.php` + partials
4) Convert home + contact views
5) Start session; add flash partial
6) Update controller to use `flash()` + `redirect()`
7) Test GET/POST + PRG

---

# Common Pitfalls
- Forgetting to start the session for flash
- Missing `end()` after `start()`
- Not escaping user/DB data
- Rendering without setting layout/section
- Skipping PRG (re-post on refresh)

---

# References
- `src/Support/View.php`
- `src/Views/layouts/main.php`
- `src/Views/partials/*`
- `src/Controllers/ContactController.php`
