import requests from bs4 import BeautifulSoup from urllib.parse import urljoin

Basic payloads for testing (non-destructive)

SQL_PAYLOADS = ["' OR '1'='1", "" OR "1"="1"] XSS_PAYLOADS = ["<script>alert('XSS')</script>"]

visited_links = set()

def get_all_links(url): """Extract all links from a page""" links = [] try: response = requests.get(url, timeout=5) soup = BeautifulSoup(response.text, "html.parser")

for a_tag in soup.find_all("a"):
        href = a_tag.get("href")
        if href:
            full_url = urljoin(url, href)
            if full_url.startswith(url):
                links.append(full_url)
except Exception as e:
    print(f"[!] Error crawling {url}: {e}")
return links

def extract_forms(url): """Extract forms from a page""" try: response = requests.get(url, timeout=5) soup = BeautifulSoup(response.text, "html.parser") return soup.find_all("form") except: return []

def get_form_details(form): """Extract form details""" details = {} action = form.attrs.get("action") method = form.attrs.get("method", "get").lower()

inputs = []
for input_tag in form.find_all("input"):
    input_type = input_tag.attrs.get("type", "text")
    input_name = input_tag.attrs.get("name")
    inputs.append({"type": input_type, "name": input_name})

details["action"] = action
details["method"] = method
details["inputs"] = inputs
return details

def submit_form(form_details, url, payload): """Submit form with payload""" target_url = urljoin(url, form_details["action"]) data = {}

for input in form_details["inputs"]:
    if input["name"]:
        if input["type"] == "text" or input["type"] == "search":
            data[input["name"]] = payload
        else:
            data[input["name"]] = "test"

try:
    if form_details["method"] == "post":
        return requests.post(target_url, data=data)
    else:
        return requests.get(target_url, params=data)
except:
    return None

def scan_sql_injection(url): """Check for SQL Injection vulnerabilities""" print(f"[+] Testing SQL Injection on: {url}") forms = extract_forms(url)

for form in forms:
    details = get_form_details(form)
    for payload in SQL_PAYLOADS:
        response = submit_form(details, url, payload)
        if response and "error" in response.text.lower():
            print(f"[!] Possible SQL Injection detected at {url}")
            print(f"    Payload: {payload}")

def scan_xss(url): """Check for XSS vulnerabilities""" print(f"[+] Testing XSS on: {url}") forms = extract_forms(url)

for form in forms:
    details = get_form_details(form)
    for payload in XSS_PAYLOADS:
        response = submit_form(details, url, payload)
        if response and payload in response.text:
            print(f"[!] Possible XSS detected at {url}")
            print(f"    Payload: {payload}")

def crawl_and_scan(url): """Crawl the website and scan pages""" if url in visited_links: return

visited_links.add(url)
print(f"\n[+] Crawling: {url}")

scan_sql_injection(url)
scan_xss(url)

links = get_all_links(url)
for link in links:
    crawl_and_scan(link)

if name == "main": target = input("Enter target URL (e.g., http://example.com): ") crawl_and_scan(target)
