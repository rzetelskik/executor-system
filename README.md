<section id="distributed-systems-large-assignment-1" class="content">
<h1>Distributed Systems Large Assignment 1</h1>
<h3 id="executor-system">Executor system</h3>
<p>Your task is to implement an executor system supporting arbitrary user-provided types of modules and messages. The solution shall take the form of a library, and follow the asynchronous programming paradigm. A template and public tests are provided in <a href="./dsassignment1.tgz">this package</a>.</p>
<h3 id="executor-system-specification">Executor system specification</h3>
<p>Your solution shall have the same public interface as the one provided in <code>solution/lib.rs</code>, except for adding fields to the <code>System</code> and <code>ModuleRef</code> structs. You can add private items and change the file structure.</p>
<p>The executor system interface provides the following functionality:</p>
<ul>
<li>Creating and starting new instances of the system (<code>System::new()</code>).</li>
<li>Registering modules in the system (<code>System::register_module()</code>). A module must satisfy the <code>Send + &#39;static</code> trait bounds. Registering a module yields a <code>ModuleRef</code>, which can be used then to send messages to the module.</li>
<li>Sending messages to registered modules (<code>ModuleRef::send()</code>). A message of type <code>M</code> can be sent to a module of type <code>T</code> if <code>T</code> implements the <code>Handler&lt;M&gt;</code> trait.</li>
<li>Creating new references to registered modules (<code>&lt;ModuleRef as Clone&gt;::clone()</code>).</li>
<li>Scheduling a <code>Tick</code> message to be sent to a registered module periodically with a given interval (<code>System::request_tick()</code>). The first tick should be sent immediately.</li>
<li>Shutting the system down gracefully (<code>System::shutdown()</code>).</li>
</ul>
<p>The <code>Message</code> trait defines what is expected from messages in the executor system. It must be also possible to use <code>ModuleRef</code> as <code>Message</code>.</p>
<p>The file <code>public-tests/tests/executors.rs</code> contains an example of how the executor system can be used. In the example there are two modules: <code>Ping</code> and <code>Pong</code>. After being registered in the system, they are sent references to each other so that they can communicate. Then one of the modules sends a <code>Ball</code> message to the other module. That module replies to it with a new <code>Ball</code> message, to which the first module replies with the next <code>Ball</code> message, and so forth.</p>
<h4 id="additional-requirements">Additional requirements</h4>
<p>You do not need to remove a module from the system when its last <code>ModuleRef</code> is dropped.</p>
<p>Ticks requested by <code>System::request_tick()</code> must be delivered at specified time intervals. There shall be no drift. When averaged over a large time interval, the frequency of ticks shall be the inverse of their specified interval.</p>
<p>The executor system must not panic unless a registered module panics (because of a user-provided message handler).</p>
<p>Your solution should be <strong>asynchronous</strong> and allow multiple modules to execute <strong>concurrently</strong>. It will be run using Tokio. Keep in mind you can run as many Tokio tasks as you want.</p>
<h3 id="varia">Varia</h3>
<p>You can use logging if you want to, but do not emit a large amount of logs at levels <code>&gt;= INFO</code> when the system is operating properly.</p>
<p>You can only use crates specified in the provided <code>Cargo.toml</code> file.</p>
<h3 id="hints">Hints</h3>
<p>You might encounter the following problem: what type to use for “some message that can be handled by a module of type <code>T</code>”? One way of dealing with this issue is to introduce a private helper trait:</p>
<div class="sourceCode" id="cb1"><pre class="sourceCode numberSource rust numberLines"><code class="sourceCode rust"><span id="cb1-1"><a href="#cb1-1"></a><span class="at">#[</span><span class="pp">async_trait::</span>async_trait<span class="at">]</span></span>
<span id="cb1-2"><a href="#cb1-2"></a><span class="kw">trait</span> Handlee<span class="op">&lt;</span>T<span class="op">&gt;:</span> <span class="bu">Send</span> <span class="op">+</span> <span class="ot">&#39;static</span></span>
<span id="cb1-3"><a href="#cb1-3"></a><span class="kw">where</span></span>
<span id="cb1-4"><a href="#cb1-4"></a>    T<span class="op">:</span> <span class="bu">Send</span><span class="op">,</span></span>
<span id="cb1-5"><a href="#cb1-5"></a><span class="op">{</span></span>
<span id="cb1-6"><a href="#cb1-6"></a>    <span class="kw">async</span> <span class="kw">fn</span> get_handled(<span class="kw">self</span><span class="op">:</span> <span class="dt">Box</span><span class="op">&lt;</span><span class="dt">Self</span><span class="op">&gt;,</span> module<span class="op">:</span> <span class="op">&amp;</span><span class="kw">mut</span> T)<span class="op">;</span></span>
<span id="cb1-7"><a href="#cb1-7"></a><span class="op">}</span></span>
<span id="cb1-8"><a href="#cb1-8"></a></span>
<span id="cb1-9"><a href="#cb1-9"></a><span class="at">#[</span><span class="pp">async_trait::</span>async_trait<span class="at">]</span></span>
<span id="cb1-10"><a href="#cb1-10"></a><span class="kw">impl</span><span class="op">&lt;</span>M<span class="op">,</span> T<span class="op">&gt;</span> Handlee<span class="op">&lt;</span>T<span class="op">&gt;</span> <span class="kw">for</span> M</span>
<span id="cb1-11"><a href="#cb1-11"></a><span class="kw">where</span></span>
<span id="cb1-12"><a href="#cb1-12"></a>    T<span class="op">:</span> Handler<span class="op">&lt;</span>M<span class="op">&gt;</span> <span class="op">+</span> <span class="bu">Send</span><span class="op">,</span></span>
<span id="cb1-13"><a href="#cb1-13"></a>    M<span class="op">:</span> Message<span class="op">,</span></span>
<span id="cb1-14"><a href="#cb1-14"></a><span class="op">{</span></span>
<span id="cb1-15"><a href="#cb1-15"></a>    <span class="kw">async</span> <span class="kw">fn</span> get_handled(<span class="kw">self</span><span class="op">:</span> <span class="dt">Box</span><span class="op">&lt;</span><span class="dt">Self</span><span class="op">&gt;,</span> module<span class="op">:</span> <span class="op">&amp;</span><span class="kw">mut</span> T) <span class="op">{</span></span>
<span id="cb1-16"><a href="#cb1-16"></a>        module<span class="op">.</span>handle(<span class="op">*</span><span class="kw">self</span>)<span class="op">.</span><span class="kw">await</span></span>
<span id="cb1-17"><a href="#cb1-17"></a>    <span class="op">}</span></span>
<span id="cb1-18"><a href="#cb1-18"></a><span class="op">}</span></span></code></pre></div>
<p>You can then use a trait object of type <code>Box&lt;dyn Handlee&lt;T&gt;&gt;</code>. The traits <code>Handler</code> and <code>Handlee</code> are related in a way similar to <code>From</code> and <code>Into</code>. An important difference is that here we use <code>self: Box&lt;Self&gt;</code>. Thanks to that when calling <code>get_handled()</code> we can move the box rather than the boxed value. The latter would be impossible, because in <code>Box&lt;dyn Handlee&lt;T&gt;&gt;</code> the boxed value has a statically unknown size.</p>
<h3 id="testing">Testing</h3>
<p>You are given a subset of official tests. Their intention is to make sure that the public interface of your solution is correct, and to evaluate basic functionality.</p>
<p>Your solution will be tested with the stable Rust version <code>rustc 1.56.0 (09c42c458 2021-10-18)</code>.</p>
<h3 id="grading">Grading</h3>
<p>Your solution will be graded based on results of automatic tests and code inspection. The number of available and required points is specified in the <a href="../../">Passing Rules</a> described at the main website of the course. If your solution passes the public tests, you will receive at least the required number of points.</p>
<h3 id="asking-questions">Asking questions</h3>
<p>Questions <strong>must</strong> be asked on a dedicated Moodle forum. This way everybody will be able to read the answer. Try to ask questions early if there are any. We will try not to require any changes to existing solutions when providing answers.</p>
<h3 id="submitting-solution">Submitting solution</h3>
<p>Your solution must be submitted as a single <code>.zip</code> file with its name being your login at students (e.g., <code>ab123456.zip</code>). After unpacking the archive, a directory path named <code>ab123456/solution/</code> must be created. In the <code>solution</code> subdirectory there must be a Rust library crate that implements the required interface. Project <code>public-tests</code> must be able to be built and tested cleanly when placed next to the <code>solution</code> directory.</p>
<hr />
<p>Authors: F. Plata, K. Iwanicki, M. Banaszek, W. Ciszewski.</p>
</section>
