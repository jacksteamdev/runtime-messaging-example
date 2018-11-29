<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>runtime-messaging</title>
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__html"><h1 id="how-to-set-an-extension-icon-badge-from-a-content-script">How to set an extension icon badge from a content script</h1>
<p>Chrome Extension browser action badges can only be modified from the Extension background page, so we need a way to communicate <strong>from</strong> the content script <strong>to</strong> the background page. We can use Chrome Runtime Messaging to accomplish this.</p>
<p>The Chrome Runtime Messaging API can <a href="https://stackoverflow.com/questions/20077487/chrome-extension-message-passing-response-not-sent#comment64245056_20077854">be confusing to use</a>, so we’ll make it easier to use by wrapping our own code around it.</p>
<h2 id="sending-a-message-from-a-content-script">Sending a message from a content script</h2>
<p>For the content script, let’s wrap <code>chrome.runtime.sendMessage</code> in a Promise and add it to our injected content script:</p>
<pre class=" language-javascript"><code class="prism  language-javascript"><span class="token keyword">function</span> <span class="token function">sendMessage</span><span class="token punctuation">(</span>message<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">return</span> <span class="token keyword">new</span> <span class="token class-name">Promise</span><span class="token punctuation">(</span><span class="token punctuation">(</span>resolve<span class="token punctuation">,</span> reject<span class="token punctuation">)</span> <span class="token operator">=&gt;</span> <span class="token punctuation">{</span>
    <span class="token keyword">try</span> <span class="token punctuation">{</span>
      chrome<span class="token punctuation">.</span>runtime<span class="token punctuation">.</span><span class="token function">sendMessage</span><span class="token punctuation">(</span>message<span class="token punctuation">,</span> resolve<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span> <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">err</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
      <span class="token function">reject</span><span class="token punctuation">(</span>err<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
  <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>Before we can send a message, we need to make one. A message can be any object that can be converted into JSON, but it usually has a <code>greeting</code> property. I like to put an enum object in an <code>ENUM.js</code> file to inject along with both my content script and my background scripts, but here we’ll just use strings. We can add a <code>text</code> property to define what text to set on the browser action badge.</p>
<pre class=" language-javascript"><code class="prism  language-javascript"><span class="token keyword">const</span> message <span class="token operator">=</span> <span class="token punctuation">{</span> greeting<span class="token punctuation">:</span> <span class="token string">'SET_BADGE_TEXT'</span><span class="token punctuation">,</span> text<span class="token punctuation">:</span> <span class="token string">'😀'</span> <span class="token punctuation">}</span><span class="token punctuation">;</span>
</code></pre>
<p>Now we can send a message like this, and optionally handle the response using the <code>then</code> method.</p>
<pre class=" language-javascript"><code class="prism  language-javascript"><span class="token function">sendMessage</span><span class="token punctuation">(</span>message<span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">then</span><span class="token punctuation">(</span>response <span class="token operator">=&gt;</span> <span class="token punctuation">{</span>
    <span class="token comment">// Log the message sent back from background.js</span>
    console<span class="token punctuation">.</span><span class="token function">log</span><span class="token punctuation">(</span><span class="token string">'response from background.js:'</span><span class="token punctuation">,</span> response<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<h2 id="receiving-a-message-in-the-background-script">Receiving a message in the background script</h2>
<p>First, we need to make sure we have a background script, so add the following to your <code>manifest.json</code> file, if you don’t already have a background page or script.</p>
<pre class=" language-json"><code class="prism  language-json"><span class="token string">"background"</span><span class="token punctuation">:</span> <span class="token punctuation">{</span>
  <span class="token string">"scripts"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token string">"background.js"</span><span class="token punctuation">]</span>
<span class="token punctuation">}</span><span class="token punctuation">,</span>
</code></pre>
<p>The <code>scripts</code> property is an array of files that Chrome loads into a background page in order from first to last. They share the same global scope, so that makes using modules easy. Using the <code>import</code> keyword in Chrome Extensions can be tricky, so let’s just add a second file to our <code>scripts</code> array:</p>
<pre class=" language-json"><code class="prism  language-json"><span class="token string">"scripts"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token string">"message.bg.js"</span><span class="token punctuation">,</span> <span class="token string">"background.js"</span><span class="token punctuation">]</span>
</code></pre>
<p>Now we can create <code>message.bg.js</code> and add the following:</p>
<pre class=" language-javascript"><code class="prism  language-javascript"><span class="token keyword">function</span> <span class="token function">listenForMessage</span><span class="token punctuation">(</span>callback<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">async</span> <span class="token keyword">function</span> <span class="token function">asyncCallback</span><span class="token punctuation">(</span>message<span class="token punctuation">,</span> sender<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token comment">// Make sure the callback returns a Promise</span>
    <span class="token keyword">return</span> <span class="token function">callback</span><span class="token punctuation">(</span>message<span class="token punctuation">,</span> sender<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>

  <span class="token keyword">function</span> <span class="token function">handleMessage</span><span class="token punctuation">(</span>message<span class="token punctuation">,</span> sender<span class="token punctuation">,</span> sendResponse<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token function">asyncCallback</span><span class="token punctuation">(</span>message<span class="token punctuation">,</span> sender<span class="token punctuation">)</span>
      <span class="token punctuation">.</span><span class="token function">then</span><span class="token punctuation">(</span>result <span class="token operator">=&gt;</span>
        <span class="token function">sendResponse</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
          success<span class="token punctuation">:</span> <span class="token boolean">true</span><span class="token punctuation">,</span>
          greeting<span class="token punctuation">:</span> message<span class="token punctuation">.</span>greeting<span class="token punctuation">,</span>
          <span class="token operator">...</span>result<span class="token punctuation">,</span>
        <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
      <span class="token punctuation">)</span>
      <span class="token punctuation">.</span><span class="token keyword">catch</span><span class="token punctuation">(</span>reason <span class="token operator">=&gt;</span> <span class="token punctuation">{</span>
        <span class="token function">sendResponse</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
          success<span class="token punctuation">:</span> <span class="token boolean">false</span><span class="token punctuation">,</span>
          greeting<span class="token punctuation">:</span> message<span class="token punctuation">.</span>greeting<span class="token punctuation">,</span>
          <span class="token operator">...</span>reason<span class="token punctuation">,</span>
        <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
      <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

    <span class="token keyword">return</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>

  chrome<span class="token punctuation">.</span>runtime<span class="token punctuation">.</span>onMessage<span class="token punctuation">.</span><span class="token function">addListener</span><span class="token punctuation">(</span>handleMessage<span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>I won’t go into great detail, but at the heart of <code>listenForMessage</code> is <code>chrome.runtime.onMessage</code>, an event that fires for <code>any</code> message sent from <code>any</code> tab by <code>any</code> extension. This is important to note, so make your message <code>greeting</code> unique to your extension (<code>"cookie-magic__set-cookies"</code>).</p>
<p>We can use <code>listenForMessage</code> like this in <code>background.js</code>:</p>
<pre class=" language-javascript"><code class="prism  language-javascript"><span class="token function">listenForMessage</span><span class="token punctuation">(</span>callbackFn<span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>The first argument of <code>listenForMessage</code> is a callback that will receive the message. We can use the following as <code>callbackFn</code>:</p>
<pre class=" language-javascript"><code class="prism  language-javascript"><span class="token keyword">const</span> <span class="token function-variable function">callbackFn</span> <span class="token operator">=</span> <span class="token punctuation">(</span>message<span class="token punctuation">,</span> sender<span class="token punctuation">)</span> <span class="token operator">=&gt;</span> <span class="token punctuation">{</span>
  <span class="token keyword">switch</span> <span class="token punctuation">(</span>message<span class="token punctuation">.</span>greeting<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token keyword">case</span> <span class="token string">'SET_BADGE_TEXT'</span><span class="token punctuation">:</span>
      console<span class="token punctuation">.</span><span class="token function">log</span><span class="token punctuation">(</span><span class="token string">'message from content.js:'</span><span class="token punctuation">,</span> message<span class="token punctuation">)</span><span class="token punctuation">;</span>
      chrome<span class="token punctuation">.</span>browserAction<span class="token punctuation">.</span><span class="token function">setBadgeText</span><span class="token punctuation">(</span><span class="token punctuation">{</span> text<span class="token punctuation">:</span> message<span class="token punctuation">.</span>text <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
      <span class="token keyword">return</span> <span class="token punctuation">{</span><span class="token punctuation">}</span><span class="token punctuation">;</span>
    <span class="token keyword">default</span><span class="token punctuation">:</span> <span class="token comment">// do nothing</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre>
<p>Our callback function will take two arguments: <code>message</code> and <code>sender</code>. We know about <code>message</code>, but what is <code>sender</code>? It contains details from the browser tab that sent the message.</p>
<p>We can use a <code>switch</code> statement to check the <code>greeting</code> property and determine what to do with the message. If <code>greeting</code> is <code>SET_BADGE_TEXT</code>, we log the message to the console for the <em>background page</em> and call <code>chrome.browserAction.setBadgeText</code>, which takes an options object with one property: <code>text</code>.</p>
<p>Then we return an object with the values we want to send back to the content script. Here we don’t have anything to say to the content script, so we just send back an empty object.</p>
<p><code>listenForMessage</code> will handle sending our response for us.  It will add <code>success: true</code> to the response object if the operation didn’t throw. If there was an error, it will catch the error and send a response with <code>success: false</code> and the details of the error.</p>
<p>If our response has no <code>greeting</code>, <code>listenForMessage</code> will also add the same <code>greeting</code> value that was sent from the content script.</p>
<p>Now that the background page has a way to receive our messages, let’s try it out! Let’s add a simple button to send a message from our content script to our background page (<a href="https://github.com/jacksteamdev/runtime-messaging-example">you can view the entire project on GitHub</a>):</p>
<pre class=" language-javascript"><code class="prism  language-javascript"><span class="token keyword">const</span> button <span class="token operator">=</span> document<span class="token punctuation">.</span><span class="token function">createElement</span><span class="token punctuation">(</span><span class="token string">'button'</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
button<span class="token punctuation">.</span>innerText <span class="token operator">=</span> <span class="token string">'Set Badge Text'</span><span class="token punctuation">;</span>
button<span class="token punctuation">.</span>onclick <span class="token operator">=</span> handleClick<span class="token punctuation">;</span>
document<span class="token punctuation">.</span>body<span class="token punctuation">.</span><span class="token function">append</span><span class="token punctuation">(</span>button<span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>Load the extension into Chrome from the extensions page or reload it if you’ve already added your extension. When you reload your test page, the extension will inject the content script and add our button to the bottom of the page. When you click it, the button will send a message to your background page and change the browser action badge!</p>
</div>
</body>

</html>
