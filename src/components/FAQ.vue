<template>
    <section id="faq" class="faq theme-dark">
        <div class="container">
            <div class="card">
                <div class="card-header">
                    <h2>Frequently Asked Questions</h2>
                </div>
                <div class="card-body">
                    <details>
                        <summary>
                            How can I login with Authkey?
                        </summary>
                        <ol>
                            <li>Login to <a href="https://web.stremio.com/" target="_blank">https://web.stremio.com/</a> using your Stremio credentials in your browser.</li>
                            <li>Open the developer console (F12) and paste the following code snippet:</li>
                            <li><code>JSON.parse(localStorage.getItem("profile")).auth.key</code></li>
                            <li>Take the output value and paste it into the authkey field.</li>
                        </ol>
                    </details>
                    <details>
                        <summary>
                            Are my credentials safe?
                        </summary>
                        <p>
                            Yes. We only use your credentials to authenticate with the Stremio API. They are not stored in *ANY* way.
                        </p>
                        <p>
                            You don't have to take my word for it take a look at the source code yourself.
                        </p>
                    </details>
                    <details>
                        <summary>
                            What risks are involved?
                        </summary>
                        <p>
                            Stremio developers don't recommend fiddling with Cinemeta, use this at your own risk. If you experience any breakage, you can reset Cinemeta to its original state by using the 'Reset Cinemeta' button or by using an addon like Stremthru Sidekick.
                        </p>
                    </details>
                </div>
            </div>
        </div>
    </section>
</template>

<style scoped>
.faq.theme-dark {
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
}
.faq.theme-dark { color: var(--text); padding: 8px 16px 16px; }
.container { max-width: 960px; margin: 0 auto; }
.card { background: var(--panel); border: 1px solid var(--border); border-radius: 10px; overflow: hidden; }
.card-header { padding: 14px 16px; border-bottom: 1px solid var(--border); }
.card-body { padding: 8px 16px 16px; }
.card-body details { background: var(--panel-2); border: 1px solid var(--border); border-radius: 8px; padding: 12px 14px; margin-top: 10px; }
.card-body summary { cursor: pointer; font-weight: 600; }
.card-body a { color: #67b8ff; text-decoration: none; }
.card-body a:hover { text-decoration: underline; }
.card-body code { 
  background: #1e1e1e; 
  color: #98bad4; 
  padding: 2px 6px; 
  border-radius: 4px; 
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace; 
  font-size: 0.9em; 
  border: 1px solid var(--border); 
}
</style>
