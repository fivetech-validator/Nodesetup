# #How to Set Up Ping.Pub Explorer

buy Domains: https://www.namecheap.com/
Add Dns Domain: https://www.cloudflare.com/

Install Dependencies
Remove any existing versions of Node.js and install the required packages:



```
sudo apt autoremove nodejs -y
curl -fsSL https://deb.nodesource.com/setup_19.x | sudo -E bash -
curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | gpg --dearmor | sudo tee /usr/share/keyrings/yarnkey.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/yarnkey.gpg] https://dl.yarnpkg.com/debian stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx nodejs git yarn -y
```

NGINX Configuration
File Configuration

Create an NGINX configuration file for your explorer:

sudo nano /etc/nginx/sites-enabled/your_explorer_server.conf
Replace your_explorer_server.conf with your site's name, for example:



```
sudo nano /etc/nginx/sites-enabled/explorer.explorer.fivetech.pro
```


Create this sample configuration:


```
server {
    listen       80;
    listen  [::]:80;
    server_name explorer.fivetech.pro;

    location / {
        root /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    gzip on;
    gzip_proxied any;
    gzip_static on;
  
 }
```

Remember to replace server_name with your server's name.

SSL Configuration
Install Certificate SSL

Run Certbot to install SSL for NGINX:


```
sudo certbot --nginx --register-unsafely-without-email
```


Select option 2 and press Enter. If the BOT asks for redirection, choose YES.

After completion, you can restart NGINX:


```
sudo systemctl restart nginx
```

Explorer Configuration
Clone the Repository



```
cd $HOME
git clone https://github.com/fivetech-validator/explorer.git
```

Create or Edit Your Configuration File

```
nano $HOME/explorer/chains/testnet/airchains.json
```

Here's an example configuration:

Copy https://github.com/itrocket-am/explorer/blob/master/chains/testnet/airchains.json ( ví dụ)

```
{
  "chain_name": "airchains-testnet",
  "api": ["https://airchains-testnet-api.itrocket.net"],
  "rpc": ["https://airchains-testnet-rpc.itrocket.net"],
  "sdk_version": "0.50.3",
  "addr_prefix": "air",
  "min_tx_fee": "200",
  "logo": "/logos/airchains.jpg",
  "assets": [
    {
      "symbol": "AMF",
      "base": "amf",
      "exponent": 6,
      "logo": "/logos/airchains.jpg"
    }
  ]
}
```

Build the Explorer

Navigate to the explorer directory, install dependencies, and build:

```
cd $HOME/explorer
yarn && yarn build
```

If you encounter any issues, use the following command:

```
yarn install --ignore-engines
cd $HOME/explorer
yarn && yarn build
```

Copy the Web Files to the Nginx HTML Folder

```
sudo cp -r $HOME/explorer/dist/* /usr/share/nginx/html
sudo systemctl restart nginx
```

After completing these steps, your Ping.Pub Explorer for Crossfi should be successfully set up and accessible. Remember to replace all placeholder text (like explorer.dnsarz.xyz) with the actual values pertinent to your setup.

Edit Your Name

index.html

Sửa thông tin trong các file này

```
src/plugins/i18n/locales/en.json
```

```
src/layouts/components/NavFooter.vue
```

```
src/layouts/components/DefaultLayout.vue
```
