# Quick Start
In this section we'll cover the first steps to manage your web application or API



## 0. eQual setup
See *Installation*

## 1. Creating a package

In **public/packages/**, create a **new folder** (we'll name it "myapp")
In myapp, create a bunch of new folders named **apps**, **classes**, and **views**
Your directory should look like this :
  ..
  /packages
    /myapp
      /apps
      /classes
      /views



## 2. Defining classes

In **public/packages/myapp/classes/**, you can add a new **.class.php** file for each class you want to use. 
In this example, we define the class **Task** for a todo-list app :

*../packages/newpackage/classes/Task.class.php*

```php
<?php  
namespace myapp;
use qinoa\orm\Model; // this is a built-in object handler
    
class Task extends Model {
    public static function getColumns() {
        return array(
            'title'     => ['type' => 'string'],
            'content'   => ['type' => 'text']
        );
    }
}
```

*Note: ID key generation is handled by eQual*

But, what if we want to establish a relation between two classes, like nesting the **User** of *User.class.php* as a parameter?
That's where the types **many2many**, **one2many**, and **many2one** come in handy (*see [Understanding DBMS relationships](https://afteracademy.com/blog/what-are-the-different-types-of-relationships-in-dbms)*)

```php
// Task
// ...
      return array(
        // ...
        'user_id'	=> ['type' => 'many2one', 'foreign_object' => 'myapp\User']
      );
```
We also need to do the opposite in *User.class.php* :
```php
// User
// ...
      return array(
        // ...
        'task_id'	=> ['type' => 'one2many', 'foreign_object' => 'myapp\Task']
      );
```



## 3. Creating database tables

Now that we have classes, we want to use them in a real database
###### /!\ Work in progress ~



## 4. Changing default entry point
In **public/config/**, open *config.inc.php* (or *default.inc.php*, unrecommended) and adapt this :
```php
define('default_package', 'myapp');	// replace 'myapp' with your package's name
```
In **public/packages/myapp/**, create a *config.inc.php* file and write this :
```php
<?php
namespace config;

define('DEFAULT_APP', 'landing');	// it refers to packages/myapp/apps/landing.php
```
And you're done, eQual will now display your webapp's landing.php as default landing page





## What next?

See *Howtos* for more learning ressources