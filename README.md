# cloud-1
## Project Hosted on Google Cloud Platform

1. Create a new project.
2. Deploy a basic Wordpress webserver VM. Use a region close to you.
3. Create a Storage bucket (used for later).
4. Create a new Service Account, assign the accound roles : Cloud SQL Client and Project > Editor.
5. Create a key for the service accound and download it.
6. Reserve a Static IP. It must be Premium tier, IPv4, and type must be Global.
7. Configure Wordpress & the server
    1. SSH to the VM
        - `sudo su`
        - `cd /var/www/html`
        - Create an `.htaccess` file, add the following (Note the commented out entries for later):
            ```
            #RewriteEngine On
            #RewriteCond %{HTTP:X-Forwarded-Proto} =http
            #RewriteRule .* https://%{HTTP:Host}%{REQUEST_URI} [L,R=permanent]
            ```
        - Add the following to your `wp-config.php` file, below the Database definitions (Note the commented out entries for later):
            ```
            define( 'AS3CF_SETTINGS', serialize( array(
                'provider' => 'gcp',
                    'key-file-path' => '/etc/keyFile.json',
            ) ) );

            #define('FORCE_SSL_ADMIN', true);
            #if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false)
            #    $_SERVER['HTTPS']='on';

            ```
        - `cd /etc/ && vim keyFile.json`
        - Add the contents of the previously downloaded JSON file to `keyFile.json`
        - `cd /var/www/html/wp-content/themes/twentyseventeen/template-parts/footer`
        - `vim site-info.php`
        - Adjust the line that contains the 'printf Proudly powered by' text to look as follows:

            `printf( __( 'Proudly powered by %s<br>%s', 'twentyseventeen' ), 'WordPress', $_SERVER['SERVER_ADDR'] );`
        
            This displays the current servers internal IP in the footer, to ensure the loadbalancer is working.

    2. Login to phpMyAdmin via `ExternalIP/phpmyadmin`, using the details contained within the VM info page.
        - Open the `Wordpress` database.
        - Export the database (SQL format, Custom Export - All tables, charset: `utf8mb4`, collate: `utf8mb4_general_ci`)
        - Download the exported SQL, and upload it to your bucket.
        - Create a new Cloud SQL Instance (MySQL):
            1. Under the Configuration Options drop down: The machine type I used was `db-n1-standard-1`, SSD Disk, 25Gb in size.
            2. Under connectivity, enable Private IP.
            3. Ensure High Availability is selected under the Backups... tab
        - Create a new database, named as you please, with charset and collate settings matching the exported ones above.
        - Import the SQL backup you saved to your bucket and select your new database in the drop down.
        - Create a new user, allow them access to anyhost. Keep the username and password handy.
        - Once again, edit the `wp-config.php` file, and change the following to their relevant details:
            ```
            define( 'DB_NAME', 'Your SQL Database Name' );
            define( 'DB_USER', 'Your SQL Database Username' );
            define( 'DB_PASSWORD', 'Your SQL Database User Password' );
            define( 'DB_HOST', 'Private IP Address of SQL Instance' );
            define( 'DB_CHARSET', 'utf8mb4' );
            define( 'DB_COLLATE', 'utf8mb4_general_ci' );
            ```
    3. Login to wordpress admin via `ExternalIP/wp-admin`, using the details contained within the VM info page.
        - Install & Activate the `WP Offload Media` & `W3 Total Cache` plugins.
        - In the `WP Offload Media` settings:
            - Select Google Cloud Storage > Define Key File (already done)
            - Press next & enter the name of your previously created Storage Bucket.
        - In the `W3 Total Cache` settings:
            - In the CDN sub section of General Settings, Enable CDN and use Generic Mirror
            - Back at the top, Click on Show Required Changes and add the text displayed to your `.htaccess` file, below the commented out lines already there.
        - ONLY ONCE YOU ARE SATISFIED WITH THE WORDPRESS CONFIGURATION:
            - Change the 2 site URL fields in the General Wordpress settings to either point to your domain (if you're using one) or the load balancer IP.
    4. Finally, you can uncomment the 3 lines in each of the `.htaccess` and `wp-config.php` files that deal with HTTPS.

8. Shut down your wordpress instace.

9. Go to the `Images` tab, create a new custom image using the Disk of the VM as source.

10. Go to the `Instance Template` tab, create a new template, set the machine type to your liking, change the boot disk to a custom image type and use the one you just created, and allow HTTP / HTTPS traffic.

11. Go to the `Instance Group` tab and create a new group using the Instance template you created.
        - Auto scaling enabled , min instances : 2 , max instances : 6
        - Create health check (TCP) on port 80

12. VPC > Firewall:
    - Create a new rule:
        - Ensure Direction is Ingress
        - Targets = All Instances
        - Source Filter = IP Ranges
        - Ranges = `130.211.0.0/22, 35.191.0.0/16`
        - Specific TCP port of 80.


## If NOT using a domain (i was not):
      13. HTTP(S) Loadbalancer:
          - Create a new HTTP/S Load balancer
          - From Internet to VMs
          - Create a new backend service:
              - HTTP protocol over port 80
              - Instance group = your instance group
              - Enable Cloud CDN
              - Apply your HTTP health check.
          - Create a new frontend service:
              - HTTP protocol over port 80
              - IP address is previously reserved static IP address.


## Resources used:
- Various Google Cloud Docs
- Calvin
- https://geekflare.com/google-cloud-cdn-test/
- https://geekflare.com/gcp-load-balancer/
- https://themeisle.com/blog/install-wordpress-on-google-cloud/
- https://www.cloudbooklet.com/best-performance-wordpress-with-google-cloud-cdn-and-load-balancing/
- https://www.wpexplorer.com/install-wordpress-google-cloud/
- http://www.mindinfinity.net/autoscaling-with-google-cloud/index.html
