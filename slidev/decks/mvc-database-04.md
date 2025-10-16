---
theme: default
title: Dotenv, DB Helper, and Contact Form
info: |
  Use phpdotenv for configuration, centralize PDO, and add a secure contact flow
layout: cover
highlighter: shiki
lineNumbers: true
drawings:
  persist: false
transition: slide-left
mdc: true
download: true
exportFilename: dotenv-db-helper-contact
class: text-center
---

# Dotenv, DB Helper, and Contact Form

Secure config • Reusable PDO • Prepared statements

---
layout: two-cols
---

# Recap (Project 03)

You built:
- Front controller (`public/index.php`)
- Simple Router + routes file
- PSR-4 autoload + namespaces
- Controllers, Models, Views

::right::

# Today (Project 04)

Add:
- `vlucas/phpdotenv` for config
- `.env` and `.env.example`
- `Database` helper (PDO)
- Contact form (GET/POST)
- Prepared statements (INSERT)

---

# Why Dotenv?

Problems it solves:
- Keeps secrets out of code and VCS
- Environment-specific config (dev, prod)
- Simple key/value UX for students

Where it lives:
- `.env` next to `composer.json`
- Load early in `public/index.php`

---

# Install phpdotenv

Commands:
```bash
composer require vlucas/phpdotenv
composer dump-autoload
```

Bootstrapping:
```php
require '../vendor/autoload.php';
$dotenv = \Dotenv\Dotenv::createImmutable(dirname(__DIR__));
$dotenv->safeLoad();
$dotenv->required(['DB_HOST','DB_NAME','DB_USER','DB_PASS','DB_CHARSET'])->notEmpty();
```

Access:
```php
$host = $_ENV['DB_HOST'];
```

---

# .env vs .env.example

- `.env` – untracked, developer-specific
- `.env.example` – committed template keys
- Add `.env` to `.gitignore`

Example keys:
```
DB_HOST=db
DB_NAME=web3400
DB_USER=web3400
DB_PASS=password
DB_CHARSET=utf8mb4
```

---

# Database Helper (Design)

Goals:
- One place to configure PDO
- Read from env vars
- Reuse across controllers/models

Signature:
```php
class Database { public static function pdo(): \PDO; }
```

Attributes:
- `ERRMODE_EXCEPTION`
- `DEFAULT_FETCH_MODE => FETCH_ASSOC`
- `EMULATE_PREPARES => false`

---

# Database Helper (Code)

```php
namespace App\Support;

class Database {
  public static function pdo(): \PDO {
    $dsn = sprintf(
      'mysql:host=%s;dbname=%s;charset=%s',
      $_ENV['DB_HOST'],
      $_ENV['DB_NAME'],
      $_ENV['DB_CHARSET']
    );
    return new \PDO($dsn, $_ENV['DB_USER'], $_ENV['DB_PASS'], [
      \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
      \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
      \PDO::ATTR_EMULATE_PREPARES => false,
    ]);
  }
}
```

---

# Contact Page (Flow)

Routes:
- GET `/contact` → show form
- POST `/contact` → validate + insert

Table:
- `contact_us(id, name, email, message, submitted_at)`

Key skills:
- Server-side validation
- Prepared INSERT
- User feedback (success/error)

---

# Controller (Sketch)

```php
class ContactController extends Controller {
  public function show() { $this->render('contact', ['errors'=>[], 'old'=>[]]); }
  public function submit() {
    $name = trim($_POST['name'] ?? '');
    $email = trim($_POST['email'] ?? '');
    $message = trim($_POST['message'] ?? '');
    $errors = [];
    if ($name==='') $errors[]='Name required';
    if ($email==='' || !filter_var($email, FILTER_VALIDATE_EMAIL)) $errors[]='Valid email required';
    if ($message==='') $errors[]='Message required';
    if ($errors) return $this->render('contact', ['errors'=>$errors,'old'=>compact('name','email','message')]);
    $pdo = Database::pdo();
    $pdo->prepare('INSERT INTO contact_us (name,email,message) VALUES (:name,:email,:message)')
        ->execute([':name'=>$name, ':email'=>$email, ':message'=>$message]);
    $this->render('contact', ['errors'=>[], 'old'=>[], 'status'=>'Thanks!']);
  }
}
```

---

# View (Essentials)

```php
<form method="post" action="/contact">
  <label>Name <input name="name" required></label>
  <label>Email <input name="email" type="email" required></label>
  <label>Message <textarea name="message" required></textarea></label>
  <button type="submit">Send</button>
</form>
```

Notes:
- Echo old values after validation errors
- Escape output with `htmlspecialchars()`

---

# Prepared Statements 101

Why:
- Prevent SQL injection
- Correctly binds types and escapes inputs

Pattern:
```php
$stmt = $pdo->prepare('INSERT ... VALUES (:a, :b, :c)');
$stmt->execute([':a'=>$a, ':b'=>$b, ':c'=>$c]);
```

Avoid:
- String interpolation in SQL
- Building SQL with user input directly

---

# Demo Plan

1) Install `vlucas/phpdotenv`
2) Add `.env` + bootstrap
3) Add `Database::pdo()`
4) Build `ContactController` + routes
5) Create `contact.php` view
6) Test GET + POST; verify DB insert

---

# Common Pitfalls
- Forgetting to ignore `.env`
- Loading Dotenv after using env vars
- Wrong DSN or charset
- Not using prepared statements
- Missing HTML escaping in view

---

# DIY BaseModel (CRUD)

Why:
- Keep controllers thin and reuse CRUD logic
- Whitelist columns with `fillable`

Shape:
```php
abstract class BaseModel {
  protected static string $table;
  protected static string $primaryKey = 'id';
  protected static array $fillable = [];
  public static function find($id): ?array {}
  public static function all($limit=100,$offset=0,$orderBy=null): array {}
  public static function create(array $data): int {}
  public static function update($id, array $data): bool {}
  public static function delete($id): bool {}
}
```

Example model:
```php
final class Contact extends BaseModel {
  protected static string $table = 'contact_us';
  protected static array $fillable = ['name','email','message'];
}
```

---

# BaseModel (Code)

```php
namespace App\Models; use App\Support\Database; use PDO;
abstract class BaseModel {
  protected static string $table; protected static string $primaryKey='id';
  protected static array $fillable=[]; protected static function pdo():PDO{ return Database::pdo(); }
  public static function find($id):?array{ $s=self::pdo()->prepare('SELECT * FROM `'.static::$table.'` WHERE `'.static::$primaryKey.'`=:id LIMIT 1');$s->bindValue(':id',$id);$s->execute();$r=$s->fetch();return $r?:null; }
  public static function all(int $limit=100,int $offset=0,?string $orderBy=null):array{ $order=$orderBy?:'`'.static::$primaryKey.'` DESC'; $s=self::pdo()->prepare('SELECT * FROM `'.static::$table.'` ORDER BY '.$order.' LIMIT :l OFFSET :o'); $s->bindValue(':l',$limit,PDO::PARAM_INT); $s->bindValue(':o',$offset,PDO::PARAM_INT); $s->execute(); return $s->fetchAll(); }
  public static function create(array $data):int{ $data=array_intersect_key($data,array_flip(static::$fillable)); if(!$data) throw new \InvalidArgumentException('No fillable fields provided.'); $cols=array_keys($data); $ph=array_map(fn($c)=>':'.$c,$cols); $qc=array_map(fn($c)=>'`'.$c.'`',$cols); $sql='INSERT INTO `'.static::$table.'` ('.implode(',',$qc).') VALUES ('.implode(',',$ph).')'; $s=self::pdo()->prepare($sql); foreach($data as $c=>$v){$s->bindValue(':'.$c,$v);} $s->execute(); return (int) self::pdo()->lastInsertId(); }
  public static function update($id,array $data):bool{ $data=array_intersect_key($data,array_flip(static::$fillable)); if(!$data) return false; $sets=[]; foreach(array_keys($data) as $c){$sets[]='`'.$c.'`=:' . $c;} $sql='UPDATE `'.static::$table.'` SET '.implode(', ',$sets).' WHERE `'.static::$primaryKey.'`=:_id'; $s=self::pdo()->prepare($sql); foreach($data as $c=>$v){$s->bindValue(':'.$c,$v);} $s->bindValue(':_id',$id); return $s->execute(); }
  public static function delete($id):bool{ $s=self::pdo()->prepare('DELETE FROM `'.static::$table.'` WHERE `'.static::$primaryKey.'`=:id'); $s->bindValue(':id',$id); return $s->execute(); }
}
```

---

# Model Generator (CLI)

Script: `scripts/generate-model.php`
- Reads `INFORMATION_SCHEMA` to find PK + columns
- Skips timestamps; writes `src/Models/{Table}.php`

Usage:
```bash
php scripts/generate-model.php contact_us
```

Generated:
```php
final class Contact extends BaseModel {
  protected static string $table = 'contact_us';
  protected static string $primaryKey = 'id';
  protected static array $fillable = ['name','email','message'];
}
```

---

# Q&A / Next Steps

- PRG pattern (redirect after POST)
- CSRF token basics
- Listing messages for admin