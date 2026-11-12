---
title: "Deploying a Plotly Dash Application on University Web Servers"
teaching: 25
exercises: 10
questions:
- "How do I host a Python Plotly Dash web application on a shared Apache server without root access?"
- "How do I configure CGI routing, HTTPS headers, and Basic Authentication for Dash?"
- "What performance considerations apply when running Python applications via CGI?"
objectives:
- "Configure an isolated Conda environment for Apache CGI execution."
- "Set up `.htaccess` and a WSGI-to-CGI wrapper script (`index.cgi`)."
- "Adapt Dash URL routing prefixes (`requests_pathname_prefix`) for user sub-directories."
- "Diagnose and resolve common CGI deployment errors using SSH terminal simulation."
keypoints:
- "Apache CGI spawns a new Python process for every HTTP request; avoid heavy startup pre-loading."
- "Dash apps hosted in user directories must explicitly match `requests_pathname_prefix` (including `~username` and trailing slashes)."
- "Never use `print()` in CGI applications—standard output is reserved strictly for HTTP headers and payloads; log to `sys.stderr` instead."
- "Use `CGIPassAuth On` in `.htaccess` to prevent Apache from stripping Basic Authentication headers."
---

## Overview

Because `users.aber.ac.uk` serves personal user directories (`~/public_html`) via Apache without root privileges or background daemon permissions, standard `app.run()` development servers cannot be exposed directly. Instead, Plotly Dash applications must be wrapped as **Common Gateway Interface (CGI)** scripts using a standalone Conda environment.

:::::::::::::::::::::::::::::::::::::: callout

### How CGI Differs from Local Development
When running `app.run()` locally, Python stays resident in memory. Under **Apache CGI**, the web server starts a brand-new Python interpreter from scratch for **every** page load, layout request, and callback trigger, then terminates it once the response is sent.

::::::::::::::::::::::::::::::::::::::::::::::

---

## 1. Directory Structure

Create a `cgi-bin` directory inside your `public_html` folder to house your application, CGI entry point, and Apache routing rules:

```text
~/public_html/
└── cgi-bin/
    ├── .htaccess       # Apache routing & header configuration
    ├── index.cgi       # Executable CGI entry point
    └── app.py          # Your Plotly Dash application
    
```
## Environment Setup

Apache executes CGI scripts using a restricted system user (www-data or apache) that does not load your interactive shell profile (~/.bashrc). Create a dedicated Conda environment and reference its Python binary directly.

```bash
conda create -n dash_app python=3.11 -y
conda activate dash_app
pip install dash dash-auth
```
## Configure Apache (.htaccess)

Create ~/public_html/cgi-bin/.htaccess to enable CGI execution, allow sub-path routing (AcceptPathInfo), and prevent Apache from stripping HTTP login credentials (CGIPassAuth):

```Apache
Options +ExecCGI
AddHandler cgi-script .cgi

# Pass HTTP Basic Auth headers through to Python
CGIPassAuth On

<Files "index.cgi">
    AcceptPathInfo On
</Files>
```

## Create the Dash Application (app.py)

Standard Dash apps assume they are hosted at the root domain (/). On users.aber.ac.uk, your app lives under /~<username>/cgi-bin/index.cgi/. You must configure requests_pathname_prefix and routes_pathname_prefix so Dash's internal AJAX requests (_dash-layout, _dash-dependencies) resolve properly.

Create ~/public_html/cgi-bin/app.py (replacing <username> and <Your Name> with your details):

```python
import sys
import dash
from dash import html, dcc, Input, Output
import dash_auth

# 1. URL Prefix MUST include the tilde (~) and trailing slash (/)
URL_PREFIX = '/~<username>/cgi-bin/index.cgi/'

VALID_USERS = {
    'admin': 'change_this_password'
}

app = dash.Dash(
    __name__,
    requests_pathname_prefix=URL_PREFIX,
    routes_pathname_prefix='/',
    serve_locally=False  # Loads Dash JS/CSS from CDN to reduce CGI load
)

# Expose the underlying Flask server for the CGI runner
server = app.server

# Optional: Enable Basic Authentication
auth = dash_auth.BasicAuth(app, VALID_USERS)

app.layout = html.Div([
    html.H2("Aberystwyth Dash Deployment"),
    dcc.Dropdown(
        id='demo-dropdown',
        options=['Option A', 'Option B', 'Option C'],
        value='Option A'
    ),
    html.Div(id='output-div', style={'marginTop': '15px'}),
    
    # Mandatory University Personal Responsibility Disclaimer
    html.Footer(
        html.P(
            "The information provided on this and other pages by me, <Your Name> (<username>@aber.ac.uk), "
            "is under my own personal responsibility and not that of Aberystwyth University. "
            "Similarly, any opinions expressed are my own and are in no way to be taken as those of "
            "Aberystwyth University.",
            style={'fontSize': '12px', 'color': '#666', 'marginTop': '40px'}
        )
    )
], style={'fontFamily': 'sans-serif', 'padding': '20px'})

@app.callback(
    Output('output-div', 'children'),
    Input('demo-dropdown', 'value')
)
def update_output(val):
    return f"You selected: {val}"
```

:::::::::::::::::::::::::::::::::::::: callout
Warning: Never Use print() in CGI Scripts

Any text written to stdout via print() is injected directly into the HTTP response stream before headers are finalized. When Dash requests JSON payloads (_dash-layout), stray print() statements will corrupt the JSON and crash the frontend with a SyntaxError: JSON.parse. Always write debug logs to sys.stderr.write("message\n") instead.

::::::::::::::::::::::::::::::::::::::::::::::
## Create the CGI Entry Point (index.cgi)

Create ~/public_html/cgi-bin/index.cgi to enforce HTTPS environment variables (preventing browser Mixed Content blocks) and normalize PATH_INFO and SCRIPT_NAME for Flask/Dash:

```python
#!/aber/<username>/miniconda3/envs/dash_app/bin/python

import os
import sys

# 1. Enforce HTTPS headers to prevent browser Mixed Content blocks
os.environ['wsgi.url_scheme'] = 'https'
os.environ['HTTPS'] = 'on'
os.environ['SERVER_PORT'] = '443'
os.environ.setdefault('REQUEST_METHOD', 'GET')
os.environ.setdefault('SERVER_NAME', 'users.aber.ac.uk')

# 2. Ensure Conda site-packages and local folder are in path
sys.path.insert(0, "/aber/<username>/miniconda3/envs/dash_app/lib/python3.11/site-packages")
sys.path.insert(0, os.path.dirname(__file__))

# 3. Normalize PATH_INFO and SCRIPT_NAME for Flask/Dash routing
raw_uri = os.environ.get('REQUEST_URI', '')
if 'index.cgi' in raw_uri:
    sub_path = raw_uri.split('index.cgi', 1)[1].split('?', 1)[0]
    os.environ['PATH_INFO'] = sub_path if sub_path.startswith('/') else '/' + sub_path
    os.environ['SCRIPT_NAME'] = '/~<username>/cgi-bin/index.cgi'
else:
    os.environ['PATH_INFO'] = '/'
    os.environ['SCRIPT_NAME'] = '/~<username>/cgi-bin/index.cgi'

from wsgiref.handlers import CGIHandler
from app import server

if __name__ == '__main__':
    CGIHandler().run(server)
```

## Performance Best Practices for CGI

Because standard CGI starts a fresh Python process on every interaction:

- Avoid Heavy Startup Pre-loading: Do not load large datasets or pre-render image arrays in global scope at the top of app.py—it will re-execute on every callback click. Load individual records or slices on demand inside callbacks or Flask routes.
- Use Lightweight Asset Formats: Prefer pre-converted .png or .jpg files over on-the-fly scientific decompression (e.g., compressed DICOM or multi-channel TIFFs) whenever possible.
- Leverage Browser Caching: When serving images or static assets through custom Flask routes, attach Cache-Control headers so the browser caches them locally:

```python
response.headers["Cache-Control"] = "public, max-age=3600"
```

## Set Permissions & Verify Deployment

Apache enforces strict security permissions on CGI scripts. Directories and .cgi scripts must be executable (755), while module files and .htaccess should not be world-writable (644):

```bash
chmod 755 ~/public_html/cgi-bin
chmod 755 ~/public_html/cgi-bin/index.cgi
chmod 644 ~/public_html/cgi-bin/app.py
chmod 644 ~/public_html/cgi-bin/.htaccess
```

:::::::::::::::::::::::::::::::::::::: challenge
Exercise: Simulate an HTTP Request via SSH

Before opening your browser, test your CGI pipeline directly from the terminal to verify that Python initializes cleanly without throwing a 500 Internal Server Error.

Run a simulated GET request against your index.cgi script for the _dash-layout endpoint. What HTTP status code do you expect if Basic Authentication is enabled?

:::::::::::::::: solution

Solution

Run the following command in your SSH session:

```bash
REQUEST_METHOD=GET REQUEST_URI="/~<username>/cgi-bin/index.cgi/_dash-layout" ~/public_html/cgi-bin/index.cgi
```
If dash_auth.BasicAuth is active and no credentials are passed in the terminal environment, you should see:

```HTTP
Status: 401 UNAUTHORIZED
WWW-Authenticate: Basic realm="User Visible Realm"
Content-Type: text/html; charset=utf-8
```
If authentication is disabled, you will see Status: 200 OK followed by the JSON layout definition. Any Python Traceback printed here indicates a syntax, import, or path error in app.py or index.cgi.

:::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::
Browser Access

Navigate to the full URL including the trailing slash:


```text
[https://users.aber.ac.uk/~](https://users.aber.ac.uk/~)<username>/cgi-bin/index.cgi/
```

## Troubleshooting Reference

| Symptom | Root Cause | Fix |
| :--- | :--- | :--- |
| **`404 Not Found` on layout** | Missing trailing slash `/` in the URL or missing `~` in `URL_PREFIX`. | Always visit `.../index.cgi/` and match `requests_pathname_prefix` exactly. |
| **`JSON.parse` SyntaxError** | Calling `print()` writes text to `stdout`, corrupting Dash's JSON responses. | Remove all `print()` calls; write debug logs to `sys.stderr.write()`. |
| **Infinite Login Prompt Loop** | Apache strips the `Authorization` header on internal AJAX requests. | Add `CGIPassAuth On` to `.htaccess`. |
| **Broken images on custom routes** | `dash-auth` blocks image routes because `<img>` tags omit auth headers. | Subclass `dash_auth.BasicAuth` and override `is_authorized()` to whitelist image paths. |
| **`500 Internal Server Error`** | Insecure permissions (`777`) or invalid shebang path in `index.cgi`. | Run `chmod 755` on scripts/folders and test via SSH terminal simulation. |

