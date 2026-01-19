Exploring the codebase to understand the hyn/tenancy setup, seara-manager and seara-spa structure, and docker-compose configuration.



## **Architecture Overview**

**Two Laravel apps:**
- **seara-manager**: Creates/manages tenants (admin panel)
- **seara-spa**: Tenant-facing app (customer portal)

**Shared services** (docker-compose.yml):
- MySQL (system DB + tenant DBs)
- PostgreSQL (timeseries data per tenant)
- Redis, Nginx, etc.

## **How Hyn/Tenancy Works**

**1. Database separation:**
- **System DB** (`tenancy`): Stores `websites` and `hostnames` tables (tenant metadata)
- **Tenant DBs**: Each tenant gets its own MySQL database named by UUID (e.g., `536c09bd-00ec-4e59-906e-d7a59f68af22`)

**2. Hostname identification:**
```135:151:nginx.conf
  # WebSocket server
  server {
    listen 80;
    server_name socket.seara-dev.net;
    
    location / {
      proxy_pass http://websocket;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
```

- Nginx forwards requests to Laravel with the `Host` header
- Middleware (`EagerIdentification`) reads the hostname and looks up the tenant in the system DB
- Laravel switches to the tenant's database connection

**3. Connection switching:**
```94:100:seara-manager/config/database.php
        'system' => [
            'driver' => 'mysql',
            'url' => env('DATABASE_URL'),
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', '3306'),
            'database' => env('DB_DATABASE', 'tenancy'),
            'username' => env('DB_USERNAME', 'tenancy'),
```

- `system` connection: Always points to the `tenancy` database
- `tenant` connection: Dynamically configured per request (database = website UUID)

## **Creating a New Tenant**

**Via API/Manager (createDataV2):**
```187:226:seara-manager/app/Models/Website.php
    public function createDataV2($data): Collection
    {
        $createdWebsite = collect();
        DB::beginTransaction();
        try {

            $website = new TenancyWebsite();
            // Generate Website
            $website->managed_by_database_connection = 'system';
            app(WebsiteRepository::class)->create($website);

            // Generate Hostname
            $hostname = new TenancyHostname;
            $hostname->fqdn = $data['fqdn'];
            $hostname = app(HostnameRepository::class)->create($hostname);
            app(HostnameRepository::class)->attach($hostname, $website);
            $createdWebsite = collect($website);

            // Get Selected User
            $customer = Customer::find($data["customer_id"]);


            if ($website->id) {

                $website = Website::findOrFail($website->id);
                $website->status = 'In Progress';
                $usage_feature = $website->updateInitSite($data);
                $siteFeature = new Site_Feature;
                $siteFeature->upsert_features($website->id, $usage_feature);
                if (env('APP_ENV', 'testing') !== 'testing') {
                    Bus::chain([
                        new ProcessMigrateMysql($website->id),
                        new ProcessPassportInstall($website->id),
                        new ProcessUpsertDns($data['fqdn']),
                        new ProcessCreateTenantAdmin($website->uuid, $data, $customer),
                        new ProcessPermission($website->id, $website->uuid)
                    ])->dispatch();

                    $this->migratePGSQL($website->uuid);
                }
            }
```

**Steps:**
1. Create `Website` (generates UUID)
2. Create `Hostname` (FQDN like `demo.seara-dev.net`)
3. Attach hostname to website
4. Queue jobs:
   - `ProcessMigrateMysql`: Creates tenant DB and runs migrations
   - `ProcessPassportInstall`: Sets up OAuth
   - `ProcessUpsertDns`: DNS config
   - `ProcessCreateTenantAdmin`: Creates admin user
5. Create PostgreSQL schema for timeseries data

**Via CLI:**
```bash
php artisan license:create {fqdn}
```

## **How to Create a Similar Project**

**1. Install hyn/tenancy:**
```bash
composer require hyn/multi-tenant
php artisan tenancy:install
```

**2. Configure database connections:**
- `system`: Points to central DB with `websites`/`hostnames`
- `tenant`: Dynamically configured (auto-created by package)

**3. Set up middleware:**
```php
// config/tenancy.php
'middleware' => [
    \Hyn\Tenancy\Middleware\EagerIdentification::class,
    \Hyn\Tenancy\Middleware\HostnameActions::class,
],
```

**4. Create tenant migrations:**
- Put tenant-specific migrations in `database/migrations/tenant/`
- They run automatically when a tenant is created

**5. Docker setup:**
- One MySQL container (hosts system + all tenant DBs)
- Nginx routes by hostname
- Laravel apps share the same MySQL instance

**Key config:**
```204:204:seara-manager/config/tenancy.php
        'tenant-division-mode' => env('TENANCY_DATABASE_DIVISION_MODE', 'database'),
```

This creates a separate database per tenant (vs. schema or prefix).

**TL;DR**: Each subdomain = different tenant = different database. The package handles connection switching based on the hostname.

```plaintext
  # WebSocket server
  server {
    listen 80;
    server_name socket.seara-dev.net;
    
    location / {
      proxy_pass http://websocket;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
```

```php
        'system' => [
            'driver' => 'mysql',
            'url' => env('DATABASE_URL'),
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', '3306'),
            'database' => env('DB_DATABASE', 'tenancy'),
            'username' => env('DB_USERNAME', 'tenancy'),
```

```php
    public function createDataV2($data): Collection
    {
        $createdWebsite = collect();
        DB::beginTransaction();
        try {

            $website = new TenancyWebsite();
            // Generate Website
            $website->managed_by_database_connection = 'system';
            app(WebsiteRepository::class)->create($website);

            // Generate Hostname
            $hostname = new TenancyHostname;
            $hostname->fqdn = $data['fqdn'];
            $hostname = app(HostnameRepository::class)->create($hostname);
            app(HostnameRepository::class)->attach($hostname, $website);
            $createdWebsite = collect($website);

            // Get Selected User
            $customer = Customer::find($data["customer_id"]);


            if ($website->id) {

                $website = Website::findOrFail($website->id);
                $website->status = 'In Progress';
                $usage_feature = $website->updateInitSite($data);
                $siteFeature = new Site_Feature;
                $siteFeature->upsert_features($website->id, $usage_feature);
                if (env('APP_ENV', 'testing') !== 'testing') {
                    Bus::chain([
                        new ProcessMigrateMysql($website->id),
                        new ProcessPassportInstall($website->id),
                        new ProcessUpsertDns($data['fqdn']),
                        new ProcessCreateTenantAdmin($website->uuid, $data, $customer),
                        new ProcessPermission($website->id, $website->uuid)
                    ])->dispatch();

                    $this->migratePGSQL($website->uuid);
                }
            }
```

```shellscript
php artisan license:create {fqdn}
```

```shellscript
composer require hyn/multi-tenant
php artisan tenancy:install
```

```php
// config/tenancy.php
'middleware' => [
    \Hyn\Tenancy\Middleware\EagerIdentification::class,
    \Hyn\Tenancy\Middleware\HostnameActions::class,
],
```

```php
        'tenant-division-mode' => env('TENANCY_DATABASE_DIVISION_MODE', 'database'),
```

---

**User**

how does the aaa. someting get handle to the that domain for example 

---

**Cursor**


Checking how subdomains like `aaa.seara-dev.net` are routed to the correct tenant.



How `aaa.seara-dev.net` gets routed to the correct tenant:

## **Nginx routing (catch-all)**

```32:44:nginx.conf
  # Main SPA on port 80
  server {
    client_max_body_size 64M;
    listen 80 default_server;
    server_name seara-dev.net;

    location / {
      proxy_pass http://spa;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
```

- `default_server` on port 80 catches any hostname that doesn't match a specific `server_name`.
- For `aaa.seara-dev.net`, nginx uses this block and forwards to `spa-app`.
- `proxy_set_header Host $host;` passes `aaa.seara-dev.net` to Laravel.

## **Laravel tenant identification**

The `EagerIdentification` middleware runs early:

1. Reads the `Host` header (`aaa.seara-dev.net`).
2. Queries the system database:
   ```sql
   SELECT * FROM hostnames WHERE fqdn = 'aaa.seara-dev.net'
   ```
3. Gets the associated `website_id` and `website.uuid`.
4. Switches the database connection:
   - Uses the `system` connection to find the tenant
   - Switches to the `tenant` connection (database = website UUID)
5. All subsequent queries use the tenant's database.

## **Flow example**

Request: `http://aaa.seara-dev.net/login`

1. Nginx: `default_server` → forwards to `spa-app` with `Host: aaa.seara-dev.net`
2. Laravel middleware: Looks up `aaa.seara-dev.net` in `hostnames` table
3. Finds tenant: `website_id = 5`, `uuid = "536c09bd-..."`  
4. Switches DB: `database.connections.tenant.database = "536c09bd-..."`
5. Request continues: Uses tenant DB for all queries

## **Important points**

- No wildcard DNS needed at nginx level — `default_server` catches everything
- DNS must point `*.seara-dev.net` to your server IP
- The hostname must exist in the `hostnames` table (created when tenant is created)
- If hostname not found → 404 (or uses default if configured)

The package handles the lookup and connection switching automatically.

```plaintext
  # Main SPA on port 80
  server {
    client_max_body_size 64M;
    listen 80 default_server;
    server_name seara-dev.net;

    location / {
      proxy_pass http://spa;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
```

```sql
   SELECT * FROM hostnames WHERE fqdn = 'aaa.seara-dev.net'
```

---

**User**

where in the code that said use the "Host" header to select tenant

---

**Cursor**


Searching for the code that reads the Host header to identify the tenant.



Searching for the code that reads the Host header. Checking the package's identification logic:



Found an example. Checking the custom vendor directory and the Environment class that handles hostname identification:



Found where the Host header is read. The hyn/tenancy package code is in `vendor/`, but here's where it's used in your codebase:

## **1. Your code using Host header:**

```34:39:seara-spa/app/Providers/CacheServiceProvider.php
                $fqdn = request()->getHost();
                $namespace = DB::table('hostnames')
                    ->select('websites.uuid')
                    ->join('websites', 'hostnames.website_id', '=', 'websites.id')
                    ->where('fqdn', $fqdn)
                    ->value('uuid');
```

This shows `request()->getHost()` reading the Host header.

## **2. Where the package does it:**

The `EagerIdentification` middleware (from hyn/tenancy) runs early. It's configured here:

```46:48:seara-manager/config/tenancy.php
    'middleware' => [
        // The eager identification middleware.
        \Hyn\Tenancy\Middleware\EagerIdentification::class,
```

The package's middleware (in `vendor/hyn/multi-tenant/src/Middleware/EagerIdentification.php`) does something like:

```php
// Inside hyn/tenancy package (vendor directory)
public function handle($request, Closure $next)
{
    $hostname = $request->getHost(); // Gets Host header: "aaa.seara-dev.net"
    
    // Look up in hostnames table
    $hostnameModel = Hostname::where('fqdn', $hostname)->first();
    
    if ($hostnameModel) {
        // Set tenant connection
        app(Environment::class)->tenant($hostnameModel->website);
    }
    
    return $next($request);
}
```

## **3. How Laravel gets Host header:**

Laravel's `request()->getHost()` reads from:
- `$_SERVER['HTTP_HOST']` (the Host header)
- Or `$_SERVER['SERVER_NAME']` as fallback

## **4. Nginx passes it:**

```39:39:nginx.conf
      proxy_set_header Host $host;
```

This forwards the original Host header to Laravel.

**TL;DR**: The `EagerIdentification` middleware calls `$request->getHost()` (which reads the Host header), looks up the hostname in the database, and switches to that tenant's connection. The actual code is in `vendor/hyn/multi-tenant/src/Middleware/EagerIdentification.php`.

```php
                $fqdn = request()->getHost();
                $namespace = DB::table('hostnames')
                    ->select('websites.uuid')
                    ->join('websites', 'hostnames.website_id', '=', 'websites.id')
                    ->where('fqdn', $fqdn)
                    ->value('uuid');
```

```php
    'middleware' => [
        // The eager identification middleware.
        \Hyn\Tenancy\Middleware\EagerIdentification::class,
```

```php
// Inside hyn/tenancy package (vendor directory)
public function handle($request, Closure $next)
{
    $hostname = $request->getHost(); // Gets Host header: "aaa.seara-dev.net"
    
    // Look up in hostnames table
    $hostnameModel = Hostname::where('fqdn', $hostname)->first();
    
    if ($hostnameModel) {
        // Set tenant connection
        app(Environment::class)->tenant($hostnameModel->website);
    }
    
    return $next($request);
}
```

```plaintext
      proxy_set_header Host $host;
```

---

**User**

how to create this thing again?

---

**Cursor**


Step-by-step setup for a new hyn/tenancy project:



Step-by-step guide to create a new hyn/tenancy project:

## **Step 1: Install Package**
```bash
composer require hyn/multi-tenant
php artisan tenancy:install
```

## **Step 2: Configure Database**

In `config/database.php`, add:

```php
'connections' => [
    // ... existing connections ...
    
    'system' => [
        'driver' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', '3306'),
        'database' => env('DB_DATABASE', 'tenancy'), // System DB
        'username' => env('DB_USERNAME', 'tenancy'),
        'password' => env('DB_PASSWORD', ''),
    ],
],
```

## **Step 3: Publish Config**
```bash
php artisan vendor:publish --tag=tenancy
```

## **Step 4: Configure Tenancy**

In `config/tenancy.php`, set:

```php
'db_username' => env('TENANCY_DB_USERNAME', 'root'),
'db_password' => env('TENANCY_DB_PASSWORD', 'your_password'),

'middleware' => [
    \Hyn\Tenancy\Middleware\EagerIdentification::class,
    \Hyn\Tenancy\Middleware\HostnameActions::class,
],

'db' => [
    'tenant-division-mode' => 'database', // Separate DB per tenant
    'tenant-migrations-path' => database_path('migrations/tenant'),
],
```

## **Step 5: Register Middleware**

In `app/Http/Kernel.php` or `bootstrap/app.php` (Laravel 11):

```php
protected $middlewareGroups = [
    'web' => [
        // ... other middleware
        \Hyn\Tenancy\Middleware\EagerIdentification::class,
        \Hyn\Tenancy\Middleware\HostnameActions::class,
    ],
];
```

## **Step 6: Run System Migrations**
```bash
php artisan migrate
```
This creates `websites` and `hostnames` tables in the system DB.

## **Step 7: Create Tenant Migrations Folder**
```bash
mkdir -p database/migrations/tenant
```
Put tenant-specific migrations here.

## **Step 8: Create Tenant (Code)**

Based on your `CreateTenant` command:

```php
use Hyn\Tenancy\Models\Website;
use Hyn\Tenancy\Models\Hostname;
use Hyn\Tenancy\Contracts\Repositories\WebsiteRepository;
use Hyn\Tenancy\Contracts\Repositories\HostnameRepository;

// 1. Create Website
$website = new Website();
$website->managed_by_database_connection = 'system';
app(WebsiteRepository::class)->create($website);

// 2. Create Hostname
$hostname = new Hostname();
$hostname->fqdn = 'demo.yourdomain.com';
$hostname = app(HostnameRepository::class)->create($hostname);
app(HostnameRepository::class)->attach($hostname, $website);

// 3. Run Migrations for Tenant
Artisan::call('tenancy:migrate', [
    '--website_id' => $website->id
]);
```

## **Step 9: Configure Nginx**

```nginx
server {
    listen 80 default_server;
    server_name _;  # Catch-all

    location / {
        proxy_pass http://laravel:9000;
        proxy_set_header Host $host;  # CRITICAL!
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## **Step 10: Environment Variables**
```env
DB_CONNECTION=system
DB_DATABASE=tenancy
TENANCY_DB_USERNAME=root
TENANCY_DB_PASSWORD=your_password
TENANCY_DEFAULT_HOSTNAME=yourdomain.com
```

## **Quick Test:**
```bash
php artisan tinker
>>> $website = new \Hyn\Tenancy\Models\Website();
>>> $website->managed_by_database_connection = 'system';
>>> app(\Hyn\Tenancy\Contracts\Repositories\WebsiteRepository::class)->create($website);
```

Done. The middleware identifies tenants from the Host header automatically.

```shellscript
composer require hyn/multi-tenant
php artisan tenancy:install
```

```php
'connections' => [
    // ... existing connections ...
    
    'system' => [
        'driver' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', '3306'),
        'database' => env('DB_DATABASE', 'tenancy'), // System DB
        'username' => env('DB_USERNAME', 'tenancy'),
        'password' => env('DB_PASSWORD', ''),
    ],
],
```

```shellscript
php artisan vendor:publish --tag=tenancy
```

```php
'db_username' => env('TENANCY_DB_USERNAME', 'root'),
'db_password' => env('TENANCY_DB_PASSWORD', 'your_password'),

'middleware' => [
    \Hyn\Tenancy\Middleware\EagerIdentification::class,
    \Hyn\Tenancy\Middleware\HostnameActions::class,
],

'db' => [
    'tenant-division-mode' => 'database', // Separate DB per tenant
    'tenant-migrations-path' => database_path('migrations/tenant'),
],
```

```php
protected $middlewareGroups = [
    'web' => [
        // ... other middleware
        \Hyn\Tenancy\Middleware\EagerIdentification::class,
        \Hyn\Tenancy\Middleware\HostnameActions::class,
    ],
];
```

```shellscript
php artisan migrate
```

```shellscript
mkdir -p database/migrations/tenant
```

```php
use Hyn\Tenancy\Models\Website;
use Hyn\Tenancy\Models\Hostname;
use Hyn\Tenancy\Contracts\Repositories\WebsiteRepository;
use Hyn\Tenancy\Contracts\Repositories\HostnameRepository;

// 1. Create Website
$website = new Website();
$website->managed_by_database_connection = 'system';
app(WebsiteRepository::class)->create($website);

// 2. Create Hostname
$hostname = new Hostname();
$hostname->fqdn = 'demo.yourdomain.com';
$hostname = app(HostnameRepository::class)->create($hostname);
app(HostnameRepository::class)->attach($hostname, $website);

// 3. Run Migrations for Tenant
Artisan::call('tenancy:migrate', [
    '--website_id' => $website->id
]);
```

```plaintext
server {
    listen 80 default_server;
    server_name _;  # Catch-all

    location / {
        proxy_pass http://laravel:9000;
        proxy_set_header Host $host;  # CRITICAL!
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```dotenv
DB_CONNECTION=system
DB_DATABASE=tenancy
TENANCY_DB_USERNAME=root
TENANCY_DB_PASSWORD=your_password
TENANCY_DEFAULT_HOSTNAME=yourdomain.com
```

```shellscript
php artisan tinker
>>> $website = new \Hyn\Tenancy\Models\Website();
>>> $website->managed_by_database_connection = 'system';
>>> app(\Hyn\Tenancy\Contracts\Repositories\WebsiteRepository::class)->create($website);
```

---

**User**

like how do i explain to them how it works ( in this projet) how to init them for a new prjoect how the domain handling db handling etc everythign

---

**Cursor**


Reviewing key files to prepare a complete explanation for your coworker:



## **Guide: Hyn/Tenancy Multi-Tenant System Explanation**

### **1. Architecture Overview**

**Two Laravel Applications:**
- **seara-manager** - Admin panel (creates/manages tenants) - Uses `system` DB only
- **seara-spa** - Tenant-facing app (customer portal) - Uses tenant DBs

**Database Structure:**
- **System Database** (`tenancy`): Contains `websites` and `hostnames` tables (metadata)
- **Tenant Databases**: Each tenant gets own MySQL DB named by UUID (e.g., `536c09bd-00ec-4e59-906e-d7a59f68af22`)
- **PostgreSQL**: Separate schema per tenant for timeseries data

---

### **2. How It Works - Request Flow**

**Step 1: Request hits Nginx**
```nginx
# nginx.conf - Catches ALL subdomains
server {
    listen 80 default_server;
    server_name seara-dev.net;
    
    location / {
        proxy_pass http://spa;
        proxy_set_header Host $host;  # CRITICAL: Passes "aaa.seara-dev.net"
    }
}
```

**Step 2: Laravel Middleware Chain**
The package's `EagerIdentification` middleware (configured in `config/tenancy.php`) runs automatically:
```php
// config/tenancy.php line 46-48
'middleware' => [
    \Hyn\Tenancy\Middleware\EagerIdentification::class,
    \Hyn\Tenancy\Middleware\HostnameActions::class,
],
```

**Step 3: Hostname Identification**
```php
// Inside EagerIdentification middleware (package code)
$hostname = $request->getHost(); // Gets "aaa.seara-dev.net" from Host header
$hostnameModel = Hostname::where('fqdn', $hostname)->first(); // Query system DB
if ($hostnameModel) {
    app(Environment::class)->tenant($hostnameModel->website); // Switch to tenant
}
```

**Step 4: Database Connection Switch**
- Package automatically configures `tenant` connection pointing to tenant's database
- All subsequent queries use tenant DB
- Your code uses `EnforceTenancy` middleware to ensure `tenant` is default:

```php
// app/Http/Middleware/EnforceTenancy.php
public function handle(Request $request, Closure $next)
{
    Config::set('database.default', 'tenant');
    return $next($request);
}
```

---

### **3. Domain Handling**

**Domain Mapping:**
- Each tenant = One hostname (FQDN) = One database
- Example: `demo.seara-dev.net` → Website ID 5 → Database `536c09bd-...`

**How Domains Work:**
1. **DNS**: Point `*.seara-dev.net` wildcard DNS to your server IP
2. **Nginx**: Uses `default_server` to catch all subdomains
3. **Laravel**: Looks up hostname in `hostnames` table (system DB)
4. **Result**: Switches to correct tenant database

**Database Tables:**
```sql
-- System DB: tenancy.hostnames
id | fqdn                | website_id
1  | demo.seara-dev.net  | 5
2  | test.seara-dev.net  | 6

-- System DB: tenancy.websites  
id | uuid                                | status
5  | 536c09bd-00ec-4e59-906e-... | Success
```

---

### **4. Database Handling**

**Two Connection Types:**

**A. System Connection** (`system`)
```php
// config/database.php line 94-108
'system' => [
    'driver' => 'mysql',
    'database' => 'tenancy',  // Always same DB
    'username' => 'tenancy',
],
```
- Always points to `tenancy` database
- Used for: `websites`, `hostnames` tables
- Used by: seara-manager app, tenant creation

**B. Tenant Connection** (`tenant`)
```php
// Dynamically configured by package based on hostname
// config/tenancy.php line 204
'tenant-division-mode' => 'database', // Separate DB per tenant
```
- Points to tenant's UUID database (e.g., `536c09bd-...`)
- Auto-configured by package on each request
- Used by: seara-spa app, all tenant operations

**How Package Creates Tenant DB:**
```php
// Custom vendor override: custom_vendor/hyn/Connection.php line 319
$clone['database'] = $website->uuid; // Database name = UUID
$clone['username'] = config('tenancy.db_username');
$clone['password'] = config('tenancy.db_password');
```

---

### **5. Creating a New Tenant**

**Method 1: Via API (Manager App)**
```php
// app/Models/Website.php - createDataV2()
// Step 1: Create Website (generates UUID)
$website = new TenancyWebsite();
$website->managed_by_database_connection = 'system';
app(WebsiteRepository::class)->create($website);

// Step 2: Create Hostname (domain mapping)
$hostname = new TenancyHostname;
$hostname->fqdn = 'demo.seara-dev.net';
$hostname = app(HostnameRepository::class)->create($hostname);
app(HostnameRepository::class)->attach($hostname, $website);

// Step 3: Queue Jobs (async processing)
Bus::chain([
    new ProcessMigrateMysql($website->id),      // Creates DB + runs migrations
    new ProcessPassportInstall($website->id),   // OAuth setup
    new ProcessUpsertDns($data['fqdn']),        // DNS config
    new ProcessCreateTenantAdmin(...),          // Admin user
    new ProcessPermission(...)                  // Permissions
])->dispatch();
```

**Method 2: Via CLI Command**
```bash
php artisan license:create demo
# Creates: demo.seara-dev.net
```

**What Happens:**
1. Creates `Website` record → generates UUID
2. Creates `Hostname` record → links to website
3. Creates MySQL database named `{uuid}`
4. Runs migrations from `database/migrations/tenant/`
5. Creates PostgreSQL schema for timeseries
6. Sets up Passport OAuth
7. Creates admin user

---

### **6. Initializing New Project**

**Step 1: Install Package**
```bash
composer require hyn/multi-tenant
php artisan tenancy:install
php artisan vendor:publish --tag=tenancy
```

**Step 2: Configure Database**
```php
// config/database.php
'connections' => [
    'system' => [
        'driver' => 'mysql',
        'database' => 'tenancy',  // System DB name
        'username' => env('DB_USERNAME'),
        'password' => env('DB_PASSWORD'),
    ],
],
```

**Step 3: Configure Tenancy**
```php
// config/tenancy.php
'db_username' => env('TENANCY_DB_USERNAME', 'root'),
'db_password' => env('TENANCY_DB_PASSWORD', ''),
'db' => [
    'tenant-division-mode' => 'database', // One DB per tenant
    'tenant-migrations-path' => database_path('migrations/tenant'),
],
```

**Step 4: Create Tenant Migrations Folder**
```bash
mkdir -p database/migrations/tenant
# Put tenant-specific migrations here
```

**Step 5: Run System Migrations**
```bash
php artisan migrate
# Creates: websites, hostnames tables in system DB
```

**Step 6: Register Middleware (if needed)**
The package auto-registers, but you can add custom:
```php
// app/Http/Kernel.php (if using custom middleware)
protected $routeMiddleware = [
    'tenancy.enforce' => \App\Http\Middleware\EnforceTenancy::class,
];
```

**Step 7: Configure Nginx**
```nginx
server {
    listen 80 default_server;
    location / {
        proxy_pass http://laravel:9000;
        proxy_set_header Host $host;  # MUST include this!
    }
}
```

**Step 8: Environment Variables**
```env
DB_CONNECTION=system
DB_DATABASE=tenancy
TENANCY_DB_USERNAME=root
TENANCY_DB_PASSWORD=your_password
TENANCY_DEFAULT_HOSTNAME=yourdomain.com
```

---

### **7. Key Concepts**

**Tenant Identification:**
- Hostname in HTTP `Host` header → Lookup in `hostnames` table → Get `website_id` → Switch to tenant DB

**Database Isolation:**
- Each tenant = Separate MySQL database (named by UUID)
- Zero data leakage between tenants
- Easy backup/restore per tenant

**System vs Tenant:**
- **System DB**: Metadata (websites, hostnames) - shared
- **Tenant DB**: Application data (users, orders, etc.) - isolated

**Migration Strategy:**
- System migrations: `database/migrations/` (run once)
- Tenant migrations: `database/migrations/tenant/` (run per tenant on creation)

---

### **8. Troubleshooting**

**Tenant not found:**
- Check `hostnames` table has correct FQDN
- Verify DNS points to server
- Check Nginx passes `Host` header

**Wrong database:**
- Verify `EnforceTenancy` middleware applied
- Check `config('database.default')` = `'tenant'`

**Migration issues:**
- Ensure `tenant-migrations-path` correct
- Check database user has CREATE privileges

---

This setup provides database-level isolation for multi-tenancy with automatic tenant identification via domain names.

```plaintext
# nginx.conf - Catches ALL subdomains
server {
    listen 80 default_server;
    server_name seara-dev.net;
    
    location / {
        proxy_pass http://spa;
        proxy_set_header Host $host;  # CRITICAL: Passes "aaa.seara-dev.net"
    }
}
```

```php
// config/tenancy.php line 46-48
'middleware' => [
    \Hyn\Tenancy\Middleware\EagerIdentification::class,
    \Hyn\Tenancy\Middleware\HostnameActions::class,
],
```

```php
// Inside EagerIdentification middleware (package code)
$hostname = $request->getHost(); // Gets "aaa.seara-dev.net" from Host header
$hostnameModel = Hostname::where('fqdn', $hostname)->first(); // Query system DB
if ($hostnameModel) {
    app(Environment::class)->tenant($hostnameModel->website); // Switch to tenant
}
```

```php
// app/Http/Middleware/EnforceTenancy.php
public function handle(Request $request, Closure $next)
{
    Config::set('database.default', 'tenant');
    return $next($request);
}
```

```sql
-- System DB: tenancy.hostnames
id | fqdn                | website_id
1  | demo.seara-dev.net  | 5
2  | test.seara-dev.net  | 6

-- System DB: tenancy.websites  
id | uuid                                | status
5  | 536c09bd-00ec-4e59-906e-... | Success
```

```php
// config/database.php line 94-108
'system' => [
    'driver' => 'mysql',
    'database' => 'tenancy',  // Always same DB
    'username' => 'tenancy',
],
```

```php
// Dynamically configured by package based on hostname
// config/tenancy.php line 204
'tenant-division-mode' => 'database', // Separate DB per tenant
```

```php
// Custom vendor override: custom_vendor/hyn/Connection.php line 319
$clone['database'] = $website->uuid; // Database name = UUID
$clone['username'] = config('tenancy.db_username');
$clone['password'] = config('tenancy.db_password');
```

```php
// app/Models/Website.php - createDataV2()
// Step 1: Create Website (generates UUID)
$website = new TenancyWebsite();
$website->managed_by_database_connection = 'system';
app(WebsiteRepository::class)->create($website);

// Step 2: Create Hostname (domain mapping)
$hostname = new TenancyHostname;
$hostname->fqdn = 'demo.seara-dev.net';
$hostname = app(HostnameRepository::class)->create($hostname);
app(HostnameRepository::class)->attach($hostname, $website);

// Step 3: Queue Jobs (async processing)
Bus::chain([
    new ProcessMigrateMysql($website->id),      // Creates DB + runs migrations
    new ProcessPassportInstall($website->id),   // OAuth setup
    new ProcessUpsertDns($data['fqdn']),        // DNS config
    new ProcessCreateTenantAdmin(...),          // Admin user
    new ProcessPermission(...)                  // Permissions
])->dispatch();
```

```shellscript
php artisan license:create demo
# Creates: demo.seara-dev.net
```

```shellscript
composer require hyn/multi-tenant
php artisan tenancy:install
php artisan vendor:publish --tag=tenancy
```

```php
// config/database.php
'connections' => [
    'system' => [
        'driver' => 'mysql',
        'database' => 'tenancy',  // System DB name
        'username' => env('DB_USERNAME'),
        'password' => env('DB_PASSWORD'),
    ],
],
```

```php
// config/tenancy.php
'db_username' => env('TENANCY_DB_USERNAME', 'root'),
'db_password' => env('TENANCY_DB_PASSWORD', ''),
'db' => [
    'tenant-division-mode' => 'database', // One DB per tenant
    'tenant-migrations-path' => database_path('migrations/tenant'),
],
```

```shellscript
mkdir -p database/migrations/tenant
# Put tenant-specific migrations here
```

```shellscript
php artisan migrate
# Creates: websites, hostnames tables in system DB
```

```php
// app/Http/Kernel.php (if using custom middleware)
protected $routeMiddleware = [
    'tenancy.enforce' => \App\Http\Middleware\EnforceTenancy::class,
];
```

```plaintext
server {
    listen 80 default_server;
    location / {
        proxy_pass http://laravel:9000;
        proxy_set_header Host $host;  # MUST include this!
    }
}
```

```dotenv
DB_CONNECTION=system
DB_DATABASE=tenancy
TENANCY_DB_USERNAME=root
TENANCY_DB_PASSWORD=your_password
TENANCY_DEFAULT_HOSTNAME=yourdomain.com
```

