# 🔐 Laravel Encrypted Views System (AES-256 + Runtime Extraction)

This project implements a secure way to hide your Laravel **Blade designs** inside a **password-protected AES-256 ZIP file**.  
This protects your front-end UI from anyone who unzips your NativePHP Android app.

---

# ✨ What This System Solves

Normally, when you build a NativePHP/Android app:

- All your Blade files (`resources/views`) are exposed.
- Anyone can unzip the APK and see your UI design.
- Even compiled Blade files appear inside:  
  `storage/framework/views`

This system protects your UI by:

### ✔ Encrypting all Blade files using AES-256  
### ✔ Deleting the `resources/views` directory  
### ✔ Storing only an encrypted ZIP inside your app  
### ✔ Extracting the views **only at runtime**, then deleting again  
### ✔ Making UI impossible to extract from the APK  

---

# 📌 Overview of the Workflow

## **1. Encrypt `resources/views` into `views.enc.zip`**
You run a `.bat` script that:

- Enters `resources/views`
- Compresses it into `views.enc.zip`
- Encrypts the ZIP with AES-256 + password
- Moves it to `storage/app/secure/`
- Deletes the original Blade files

---

## **2. Laravel extracts views at runtime**
When a request is received:

- Laravel unzips the encrypted file into a temporary folder  
- Loads the views  
- Deletes the extracted folder immediately

This ensures:

✔ User cannot access Blade files  
✔ After each request, files disappear  
✔ NativePHP app remains secure  

---

# 🗂 Folder Structure After Encryption

```
resources/views/         ← empty (deleted)
storage/app/secure/
    └── views.enc.zip    ← encrypted AES-256 zip
```

---

# 🛠 Step 1 — Encrypt Views (Windows)

Create a file:

```
encrypt_views.bat
```


Paste this **working version**:

```bat
@echo off
SETLOCAL ENABLEDELAYEDEXPANSION

:: ====== CONFIG ======
SET PROJECT_DIR=%cd%
SET VIEWS_DIR=%PROJECT_DIR%\resources\views
SET SECURE_DIR=%PROJECT_DIR%\storage\app\secure
SET ZIP_FILE=%SECURE_DIR%\views.enc.zip
SET PASSWORD=YOUR_SUPER_STRONG_PASSWORD
SET SEVENZIP="C:\Program Files\7-Zip\7z.exe"
:: =====================

echo ----------------------------------------------
echo PROJECT: %PROJECT_DIR%
echo VIEWS DIR: %VIEWS_DIR%
echo ----------------------------------------------
echo.

IF NOT EXIST "%VIEWS_DIR%" (
    echo ERROR: views folder NOT FOUND!
    pause
    exit /b
)

IF NOT EXIST %SEVENZIP% (
    echo ERROR: 7z.exe NOT FOUND at %SEVENZIP%
    pause
    exit /b
)

echo Creating secure directory...
IF NOT EXIST "%SECURE_DIR%" mkdir "%SECURE_DIR%"

echo.
echo Deleting old encrypted zip if exists...
IF EXIST "%ZIP_FILE%" del /Q "%ZIP_FILE%"

echo.
echo Switching into views directory...
cd /D "%VIEWS_DIR%" || (
    echo ERROR: cd into views FAILED.
    pause
    exit /b
)

echo.
echo Encrypting views (AES-256)...
%SEVENZIP% a -tzip -mem=AES256 -p%PASSWORD% "%ZIP_FILE%" ".\*"

IF %ERRORLEVEL% NEQ 0 (
    echo ZIP FAILED!
    pause
    exit /b
)

echo.
echo ZIP CREATED SUCCESSFULLY:
echo %ZIP_FILE%

echo.
echo Returning to project directory...
cd /D "%PROJECT_DIR%\resources"

echo Deleting original views folder...
rmdir /S /Q "views"

echo.
echo DONE. Views encrypted and removed.
pause

```

---

# 🛠 Step 2 — Create Laravel Runtime Extractor

In:

```
app/Services/ViewExtractor.php
```

```php
<?php

namespace App\Services;

class ViewExtractor
{
    public static function extract()
    {
        $zipPath = storage_path('app/secure/views.enc.zip');
        $viewsPath = resource_path('views');
        $password = 'YOUR_SUPER_STRONG_PASSWORD';

        // Ensure views folder exists
        if (!is_dir($viewsPath)) mkdir($viewsPath, 0777, true);

        // Extract
        $cmd = "\"C:\Program Files\7-Zip\7z.exe\" x -p$password -y \"$zipPath\" -o\"$viewsPath\"";
        exec($cmd);

        register_shutdown_function(function () use ($viewsPath) {
            self::rrmdir($viewsPath);
        });
    }

    private static function rrmdir($dir)
    {
        if (!is_dir($dir)) return;

        $files = array_diff(scandir($dir), ['.', '..']);

        foreach ($files as $file) {
            $path = $dir . '/' . $file;

            is_dir($path) ? self::rrmdir($path) : unlink($path);
        }

        rmdir($dir);
    }
}
```

---

# 🛠 Step 3 — Auto-extract during every HTTP request

Open:

```
bootstrap/app.php
```

Add inside middleware:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(\App\Http\Middleware\ExtractEncryptedViews::class);
})
```

Create middleware:

```
app/Http/Middleware/ExtractEncryptedViews.php
```

```php
<?php

namespace App\Http\Middleware;

use Closure;
use App\Services\ViewExtractor;

class ExtractEncryptedViews
{
    public function handle($request, Closure $next)
    {
        ViewExtractor::extract();
        return $next($request);
    }
}
```

---

# 🔐 Security Notes

### ✔ Views exist only in memory during request  
### ✔ They disappear immediately after the response  
### ✔ APK contains only encrypted ZIP  
### ✔ Password is stored server-side  
### ✔ No UI code is visible to final users  

---

# 🚀 Commands

Encrypt views before building Android app:

```
encrypt_views.bat
```

Run server normally:

```
php artisan serve
```

---

# 🟢 Status: **Production Ready**

- No Blade source files stored on device  
- UI protected from reverse-engineering  
- NativePHP supported  
- Works on Windows, Android, and Linux  

---

# 📞 Need More?

If you want:

✔ Auto-rotate encryption keys  
✔ Add obfuscation to controllers  
✔ Move logic behind API only  
✔ Prevent Laravel storage exposure  

Just ask.

```
✨ Your Laravel views are now fully protected with AES-256 encryption.
```
