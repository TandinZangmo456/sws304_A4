**SWS304 – Advanced Web Attacks & Exploitations**

**Assignment 04: Prototype Pollution Labs**

# **Lab 1 — Client-Side Prototype Pollution via Browser APIs**

| Lab | Lab 1 of 4 |
| :---- | :---- |
| **Title** | Client-Side Prototype Pollution via Browser APIs |
| **Difficulty** | Practitioner |
| **Type** | Client-Side Prototype Pollution |
| **Marks** | 2.5 Marks |

## **Step 1 — Explore the Application and Find the Vulnerable JS Code**

Opening DevTools → Sources tab, the file deparam.js was identified under resources/js/. This file reads URL query parameters by splitting the query string on & and \= characters, then uses bracket notation to assign each parsed key-value pair onto a plain JavaScript object (obj). The vulnerability is in lines 37–43: when the key contains bracket notation such as \_\_proto\_\_\[testprop\], the code walks down the key path and assigns directly — there is no check to block the special key \_\_proto\_\_. This means passing ?\_\_proto\_\_\[testprop\]=polluted causes the code to do obj\['\_\_proto\_\_'\]\['testprop'\] \= 'polluted', which modifies Object.prototype itself.

*Figure 1a: DevTools Sources — deparam.js reading URL parameters*

![alt text](<ss/Screenshot from 2026-06-09 22-37-11.png>)

## **Step 2 — Inject the Payload and Verify Pollution**

**Payload used:**

?\_\_proto\_\_\[testprop\]=polluted

After appending the payload to the URL and reloading, the Console was opened and Object.prototype.testprop was evaluated. The result returned 'polluted', confirming that Object.prototype was successfully modified by the URL parameter.

*Figure 1b: Console confirming Object.prototype.testprop \= 'polluted'*

![alt text](<ss/Screenshot from 2026-06-09 22-37-28.png>)

## **Step 3 — Solve the Lab**

**Final payload URL:**

?\_\_proto\_\_\[transport\_url\]=data:,alert(1)

The searchLogger.js script (found via a search for transport\_url in Sources) reads config.transport\_url and, if it exists, creates a \<script\> element with that value as its src and appends it to the document body. By polluting transport\_url via \_\_proto\_\_, all config objects inherit the property. The data: URI causes the browser to execute alert(1), solving the lab.

*Figure 1c: alert(1) dialog triggered — XSS executed*

![alt text](<ss/Screenshot from 2026-06-09 22-37-48.png>)

*Figure 1d: Lab solved confirmation — "Congratulations, you solved the lab\!"*

![alt text](<ss/Screenshot from 2026-06-09 22-38-10.png>)

## **Summary — Lab 1**

The lab exploited a client-side prototype pollution vulnerability in the deparam.js browser library. The library parsed URL query parameters directly into a JavaScript object without filtering the special key \_\_proto\_\_, so an attacker could inject arbitrary properties into Object.prototype. Since all plain JavaScript objects inherit from Object.prototype, every object on the page inherited the injected property. This was chained with a DOM sink (searchLogger.js) that used the polluted transport\_url property to dynamically load a script, leading to JavaScript execution. In a real application, this could allow an attacker to hijack user sessions, steal credentials, or perform actions on behalf of the victim simply by tricking them into visiting a crafted URL.

## **Questions — Lab 1**

**Question 1.1**

Why does adding ?\_\_proto\_\_\[x\]=y to the URL cause Object.prototype to be modified — and not just the single object created from the URL parameters?

JavaScript objects are linked via the prototype chain. When deparam.js creates obj \= {}, that object's internal \[\[Prototype\]\] slot points to Object.prototype. When the code processes the key \_\_proto\_\_, it accesses obj\['\_\_proto\_\_'\] — which, in most engines, returns the actual Object.prototype reference (the same one shared by all plain objects). Assigning a property on that reference adds it to Object.prototype itself. Because every plain object inherits from the same Object.prototype, every newly created {} on the page (and many existing ones) immediately sees the injected property as if it were their own. This is not isolated to obj; it pollutes the global prototype shared by the entire JavaScript environment in that browser tab.

**Question 1.2**

Name ONE defence a developer could add to the JavaScript code to prevent this attack.

The developer should check each parsed key against a denylist before assigning it. Specifically, after extracting the key from the URL, the code should verify that key \!== '\_\_proto\_\_' and key \!== 'constructor' and key \!== 'prototype' before doing any assignment. If the key matches, it is simply skipped. This stops pollution because the harmful traversal into Object.prototype is never performed — the injected key is discarded before it can be used to walk the prototype chain.

# **Lab 2 — DOM XSS via Client-Side Prototype Pollution**

| Lab | Lab 2 of 4 |
| :---- | :---- |
| **Title** | DOM XSS via Client-Side Prototype Pollution |
| **Difficulty** | Practitioner |
| **Type** | Prototype Pollution → DOM XSS |
| **Marks** | 2.5 Marks |

## **Step 1 — Find the Source and the Sink**

The Sources tab was used to locate the two key files. deparam.js serves as the pollution source — it reads URL query parameters and assigns them to an object using bracket notation, without filtering \_\_proto\_\_. The sink was found in searchLogger.js: this file calls deparam(new URL(location).searchParams) to build a config object, then checks if config.transport\_url exists. If it does, it creates a \<script\> element, sets script.src \= config.transport\_url, and appends it to document.body. This dynamic script injection is the sink — anything that ends up in config.transport\_url will be loaded as a script.

To confirm the pollution source uses URLSearchParams, a search for "URLSearchParams" in DevTools Search tab returned results pointing to deparam.js. To identify the sink property, searching for "transport\_url" highlighted the relevant lines in searchLogger.js.

**Sink property identified: transport\_url**

*Figure 2a: Console — Object.prototype showing foo: 'bar' after pollution test (URLSearchParams search confirms source)*

![alt text](<ss/Screenshot from 2026-06-09 22-38-40.png>)

*Figure 2b: Sources — searchLogger.js showing the transport\_url sink (lines 12–16 highlighted)*

![alt text](<ss/Screenshot from 2026-06-09 22-38-53.png>)

## **Step 2 — Fire the XSS**

**Payload URL:**

?\_\_proto\_\_\[transport\_url\]=data:,alert(1)

By polluting transport\_url via \_\_proto\_\_ in the URL, the config object built by searchLogger.js inherits transport\_url from Object.prototype. The condition if(config.transport\_url) evaluates to true, so the script appends a \<script src="data:,alert(1)"\> to the DOM. The browser loads the data: URI as a script, executing alert(1).

*Figure 2c: alert(1) dialog triggered — XSS executed via polluted transport\_url*

![alt text](<ss/Screenshot from 2026-06-09 22-39-06.png>)

*Figure 2d: Lab solved confirmation — "Congratulations, you solved the lab\!"*

![alt text](<ss/Screenshot from 2026-06-09 22-39-17.png>)

## **Summary — Lab 2**

This lab demonstrated a two-stage attack. In stage one, the URL query parameter parser (deparam.js) was exploited to inject the value data:,alert(1) into Object.prototype under the key transport\_url. In stage two, the searchLogger.js script created a config object from the same URL parameters; because the object inherits from the polluted Object.prototype, config.transport\_url resolved to the injected value even though the URL contained no explicit transport\_url parameter. The script then appended a dynamic \<script\> element with that value as its src, causing the browser to execute the attacker-controlled JavaScript. This chain — pollution source to inherited property to DOM sink — shows how prototype pollution can be elevated to full script execution without the attacker ever directly touching the vulnerable sink.

## **Questions — Lab 2**

**Question 2.1**

Why would injecting the XSS payload directly into the sink (without prototype pollution) NOT work in this lab?

The sink reads config.transport\_url, where config is built by calling deparam() on the URL's search parameters. The deparam function only sets own properties on the resulting object — it would need an explicit URL parameter named transport\_url (e.g. ?transport\_url=...) to set that key directly. However, the application likely filters or ignores a plainly-named transport\_url parameter, or the sink is only reached when the property is inherited (not explicitly set). Prototype pollution bypasses this because the injected value lives on Object.prototype, not on the config object itself; when JavaScript looks up config.transport\_url and finds no own property, it walks up the prototype chain and finds the polluted value there. Without the pollution step, there is no mechanism to place a value on the prototype chain from the URL.

**Question 2.2**

A developer adds DOMPurify to sanitise all innerHTML assignments. Does this fully fix the vulnerability?

No, it does not fully fix the vulnerability. DOMPurify sanitises HTML strings before they are written to innerHTML — it would prevent malicious HTML from being injected via that specific sink. However, the sink in this lab is not innerHTML: it is dynamic script loading via script.src. DOMPurify has no effect on that sink because the browser fetches and executes whatever URL is placed in script.src before any sanitisation step would apply. To fully fix the vulnerability, two things must be done: first, the prototype pollution source (deparam.js) must be patched to block the \_\_proto\_\_ key, so attacker-controlled values cannot reach Object.prototype in the first place. Second, the searchLogger.js sink must be hardened — for example, by using Object.create(null) for the config object (so it has no prototype and cannot inherit polluted values), or by explicitly validating that transport\_url is a known, trusted URL before using it as a script source.

# **Lab 3 — Privilege Escalation via Server-Side Prototype Pollution**

| Lab | Lab 3 of 4 |
| :---- | :---- |
| **Title** | Privilege Escalation via Server-Side Prototype Pollution |
| **Difficulty** | Practitioner |
| **Type** | Server-Side Prototype Pollution |
| **Marks** | 2.5 Marks |

## **Step 1 — Intercept the JSON Request in Burp Suite**

After logging in as the regular user "wiener", the address update form under My Account was submitted. Burp Suite intercepted the outgoing POST request to /my-account/change-address. The request body was a JSON object containing the address fields (address\_line\_1, address\_line\_2, city, postcode, country) and a sessionId. The server responded with the updated user object, which included "isAdmin": false, confirming the account was a non-privileged user.

*Figure 3a: Burp Suite — original POST /my-account/change-address JSON request body*

![alt text](<ss/Screenshot from 2026-06-09 22-39-48.png>)

## **Step 2 — Inject the \_\_proto\_\_ Payload**

The intercepted request was sent to Repeater. The JSON body was modified by appending a \_\_proto\_\_ key. First, a test payload was sent to confirm the server was vulnerable:

"\_\_proto\_\_": { "foo": "bar" }

The server response included "foo": "bar" in the returned user object — this confirmed that the server's merge function was copying \_\_proto\_\_ properties onto the response/user object, indicating prototype pollution. With the vulnerability confirmed, the payload was changed to escalate privileges:

"\_\_proto\_\_": { "isAdmin": true }

The server responded with "isAdmin": true in the user object, confirming that the pollution had succeeded and the user's effective role in the Node.js process was now admin.

*Figure 3b: Burp Repeater — modified request with "\_\_proto\_\_": {"foo":"bar"} and response showing foo:"bar" (pollution confirmed)*

![alt text](<ss/Screenshot from 2026-06-09 22-40-07.png>)

*Figure 3c: Burp Repeater — request with "\_\_proto\_\_": {"isAdmin":true} and server response showing "isAdmin":true*

![alt text](<ss/Screenshot from 2026-06-09 22-40-18.png>)

## **Step 3 — Solve the Lab**

With isAdmin: true now reflected in the server's user context, the page was reloaded. The navigation bar showed an Admin panel link. Navigating to the admin panel revealed a Users list with a Delete option. The user "carlos" was deleted, solving the lab.

*Figure 3d: Lab solved — "Privilege escalation via server-side prototype pollution" marked Solved; "User deleted successfully\!" message visible*

![alt text](<ss/Screenshot from 2026-06-09 22-40-43.png>)

## **Summary — Lab 3**

This lab exploited a server-side prototype pollution vulnerability in a Node.js Express application. When the POST /my-account/change-address endpoint received the JSON body, the server used a vulnerable recursive merge function to apply the submitted fields to the user's stored data. Because the merge function iterated over all keys in the JSON body without filtering \_\_proto\_\_, passing "\_\_proto\_\_": {"isAdmin": true} caused the merge to traverse into Object.prototype and set isAdmin \= true on it. Since Node.js runs a single JavaScript process shared across all requests, this pollution persisted and affected every subsequently created object in the server's memory. When the server later created a new object to represent the user's session data (e.g. via {}) and checked the isAdmin property, JavaScript found the value on Object.prototype and returned true — even though the user's own database record had isAdmin: false. This gave the attacker admin access without any change to the database.

## **Questions — Lab 3**

**Question 3.1**

After sending the \_\_proto\_\_ payload once, why did the pollution affect ALL objects in the Node.js process?

In JavaScript, every plain object literal ({}) has its internal \[\[Prototype\]\] slot pointing to a single shared Object.prototype. In a browser, this prototype is scoped to one tab's JavaScript environment. In Node.js, the entire server application runs in one shared JavaScript process — there is only one Object.prototype for the whole process. When the vulnerable merge function processed "\_\_proto\_\_": {"isAdmin": true}, it effectively executed Object.prototype.isAdmin \= true on that single shared prototype. From that moment onward, every plain object created anywhere in the server's code — including objects for other users' requests, session objects, query results, and middleware data — inherited isAdmin: true from Object.prototype via the prototype chain lookup rule: if a property is not found as an own property, JavaScript walks up the chain until it finds it or reaches null. Since no object explicitly had its own isAdmin property set to false, they all inherited the polluted value.

**Question 3.2**

What is the correct code-level fix for the vulnerable merge function on the server?

The fix is to sanitise or block dangerous keys before merging. The merge function should explicitly check that the key being processed is not \_\_proto\_\_, constructor, or prototype before assigning it. A simple example fix would be:

function safeMerge(target, source) {

  for (const key of Object.keys(source)) {

    if (key \=== "\_\_proto\_\_" || key \=== "constructor" || key \=== "prototype") continue;

    if (typeof source\[key\] \=== "object" && source\[key\] \!== null) {

      target\[key\] \= target\[key\] || {};

      safeMerge(target\[key\], source\[key\]);

    } else {

      target\[key\] \= source\[key\];

    }

  }

}

This fix blocks the pollution because the \_\_proto\_\_ key is never passed to any assignment operation — it is skipped entirely on every recursive call. A complementary fix is to use JSON.parse with a reviver that rejects these keys, or to use Object.create(null) for target so it has no prototype to pollute.