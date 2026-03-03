# PHP / MySQL / WordPress

## Overview

PHP sites (WordPress, raw PHP, Laravel as covered in 14) run server-side. MySQL connections via PHP are not affected by browser DNS poisoning. The concern is:

1. Your WordPress/PHP server's public IP being reachable through a CDN (same as any other backend)
2. PHP code making outbound requests to blocked APIs/CDNs
3. WordPress plugins making direct calls to blocked services from the browser

---

## WordPress ISP Bypass (Complete Setup)

### Problem

WordPress loads numerous external resources from the browser:
- WordPress.com stats (jetpack)
- Google Fonts (sometimes blocked)
- External plugin API endpoints
- WooCommerce payment gateways

### Solution: Cloudflare as Full Reverse Proxy

1. Add your WordPress domain to Cloudflare (free plan)
2. Set nameservers to Cloudflare's
3. Enable the Cloudflare proxy (orange cloud) for all A/CNAME records
4. Set SSL mode to **Full** (not Flexible   requires SSL on origin)

```
WordPress → yourproduct.com → Cloudflare edge (ISP cannot block *.cf IPs easily)
```

---

## Nginx + WordPress (Origin Configuration)

```nginx
# /etc/nginx/sites-available/wordpress
server {
    listen 443 ssl http2;
    server_name yourproduct.com www.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourproduct.com/privkey.pem;

    root /var/www/html;
    index index.php;

    # Security: hide server info
    server_tokens off;

    # Block direct access to wp-admin from non-Cloudflare IPs
    location /wp-admin {
        # Only allow Cloudflare IPs
        allow 103.21.244.0/22;
        allow 103.22.200.0/22;
        allow 104.16.0.0/13;
        allow 104.24.0.0/14;
        allow 108.162.192.0/18;
        # Add all Cloudflare IP ranges
        deny all;
        try_files $uri $uri/ /index.php?$args;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|css|js|ico|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## WordPress wp-config.php Settings

```php
<?php
// wp-config.php

// Force HTTPS
define('FORCE_SSL_ADMIN', true);

// If behind Cloudflare proxy, trust forwarded IPs
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && 
    $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}

// Database   use private IP, not hostname
define('DB_HOST', '10.0.0.2:3306');  // Private VPC IP
define('DB_NAME', 'your_db');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', getenv('DB_PASSWORD'));

// URLs   must match your CDN domain
define('WP_HOME', 'https://yourproduct.com');
define('WP_SITEURL', 'https://yourproduct.com');

// Security
define('DISALLOW_FILE_EDIT', true);
define('WP_DEBUG', false);
```

---

## WordPress Plugin: Remove External Calls

Install **WP Asset CleanUp** or manually dequeue external scripts that call blocked domains:

```php
<?php
// functions.php (theme) or a custom plugin

// Remove Google Fonts (can be blocked)
add_action('wp_enqueue_scripts', function() {
    wp_dequeue_style('google-fonts');
    
    // Self-host Google Fonts instead (download and serve from your domain)
    wp_enqueue_style('local-fonts', get_template_directory_uri() . '/fonts/fonts.css');
}, 100);

// Disable Gravatar (external call to gravatar.com)
add_filter('get_avatar', '__return_empty_string');

// Remove Jetpack external calls
add_filter('jetpack_development_mode', '__return_true');
```

---

## WooCommerce Payment Gateway Proxy

Payment gateways (Razorpay, Stripe) call their JS from external CDNs. If those are blocked:

```php
<?php
// functions.php   proxy Razorpay JS through your server
add_action('wp_enqueue_scripts', function() {
    if (is_checkout()) {
        // Instead of external:
        // wp_enqueue_script('razorpay', 'https://checkout.razorpay.com/v1/checkout.js');
        
        // Serve from your own domain (download and host locally):
        wp_enqueue_script('razorpay-local', get_template_directory_uri() . '/js/razorpay-checkout.js');
    }
});
```

---

## Raw PHP: API Proxy

```php
<?php
// proxy.php   simple PHP proxy for blocked APIs

$SUPABASE_URL = getenv('SUPABASE_URL');
$SUPABASE_KEY = getenv('SUPABASE_ANON_KEY');

$path = ltrim($_SERVER['PATH_INFO'] ?? '', '/');
$queryString = $_SERVER['QUERY_STRING'] ?? '';
$targetUrl = $SUPABASE_URL . '/' . $path . ($queryString ? '?' . $queryString : '');

$method = $_SERVER['REQUEST_METHOD'];
$body = file_get_contents('php://input');

$headers = [
    'apikey: ' . $SUPABASE_KEY,
    'Authorization: ' . ($_SERVER['HTTP_AUTHORIZATION'] ?? 'Bearer ' . $SUPABASE_KEY),
    'Content-Type: application/json',
];

$ch = curl_init($targetUrl);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);

if ($body && in_array($method, ['POST', 'PUT', 'PATCH'])) {
    curl_setopt($ch, CURLOPT_POSTFIELDS, $body);
}

$response = curl_exec($ch);
$statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);

http_response_code($statusCode);
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: ' . ($_SERVER['HTTP_ORIGIN'] ?? '*'));
echo $response;
```

---

## MySQL: Keep Database in Private Subnet

```sql
-- Create DB user restricted to private IP only
CREATE USER 'wp_user'@'10.0.0.%' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'10.0.0.%';
FLUSH PRIVILEGES;
```

Never open port 3306 to `0.0.0.0`. Use SSH tunnel or VPC private networking only.

---

## WordPress Caching for CDN Performance

```php
<?php
// functions.php   set cache headers for Cloudflare
add_action('send_headers', function() {
    if (!is_user_logged_in() && !is_admin()) {
        header('Cache-Control: public, max-age=3600, s-maxage=86400');
    } else {
        header('Cache-Control: private, no-cache, no-store');
    }
});
```

With proper caching, your WordPress site serves from Cloudflare's cache edge. Even if your origin server is temporarily unreachable, cached pages continue serving.


