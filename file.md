---


---

<h1 id="file-code-study">File code study</h1>
<p>Contents under “storage/file/”.</p>
<h2 id="relationtable-schema-index-view-...-path-and-name.-commonrelpath.c">Relation(table, schema, index, view …) path and name. common/relpath.c</h2>
<p>Under the data directory, a relation’s physical storage consists of one or more forks(names for same relation).<br>
For example a table oid “13059” under database “13213” contains files “13059”(main fork), “13059_fsm”(FSM fork), “13059_vm”(visibility fork). There may also contains a initialization fork.</p>
<pre class=" language-c"><code class="prism  language-c"><span class="token comment">/*
 * Stuff for fork names.
 *
 * The physical storage of a relation consists of one or more forks.
 * The main fork is always created, but in addition to that there can be
 * additional forks for storing various metadata. ForkNumber is used when
 * we need to refer to a specific fork in a relation.
 */</span>
<span class="token keyword">typedef</span> <span class="token keyword">enum</span> ForkNumber
<span class="token punctuation">{</span>
	InvalidForkNumber <span class="token operator">=</span> <span class="token operator">-</span><span class="token number">1</span><span class="token punctuation">,</span>
	MAIN_FORKNUM <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">,</span>
	FSM_FORKNUM<span class="token punctuation">,</span>
	VISIBILITYMAP_FORKNUM<span class="token punctuation">,</span>
	INIT_FORKNUM

	<span class="token comment">/*
	 * NOTE: if you add a new fork, change MAX_FORKNUM and possibly
	 * FORKNAMECHARS below, and update the forkNames array in
	 * src/common/relpath.c
	 */</span>
<span class="token punctuation">}</span> ForkNumber<span class="token punctuation">;</span>
<span class="token keyword">const</span> <span class="token keyword">char</span> <span class="token operator">*</span><span class="token keyword">const</span> forkNames<span class="token punctuation">[</span><span class="token punctuation">]</span> <span class="token operator">=</span> <span class="token punctuation">{</span>
	<span class="token string">"main"</span><span class="token punctuation">,</span>						<span class="token comment">/* MAIN_FORKNUM */</span>
	<span class="token string">"fsm"</span><span class="token punctuation">,</span>						<span class="token comment">/* FSM_FORKNUM */</span>
	<span class="token string">"vm"</span><span class="token punctuation">,</span>						<span class="token comment">/* VISIBILITYMAP_FORKNUM */</span>
	<span class="token string">"init"</span>						<span class="token comment">/* INIT_FORKNUM */</span>
<span class="token punctuation">}</span><span class="token punctuation">;</span>
</code></pre>
<h5 id="char--getdatabasepathoid-dbnode-oid-spcnode-construct-path-to-a-database-directory">char * GetDatabasePath(Oid dbNode, Oid spcNode): construct path to a database directory</h5>
<p>If the <code>spcNode == GLOBALTABLESPACE_OID</code>, return “global”.<br>
If <code>spcNode == DEFAULTTABLESPACE_OID</code>, return “bash/(dbNode/dboid)”.<br>
All other tablespaces are accessed via symlinks. pg_tblspc/(spcNode/spcoid)/PG_(version)/(dbNode/dboid).</p>
<h5 id="char--getrelationpathoid-dbnode-oid-spcnode-oid-relnode-int-backendid-forknumber-forknumber-construct-path-to-a-relations-file">char * GetRelationPath(Oid dbNode, Oid spcNode, Oid relNode, int backendId, ForkNumber forkNumber): construct path to a relation’s file</h5>
<p>If the <code>spcNode == GLOBALTABLESPACE_OID</code>:<br>
if <code>forkNumber != MAIN_FORKNUM</code>:<br>
return “global/reloid_forkname”<br>
else return “global/reloid”<br>
If <code>spcNode == DEFAULTTABLESPACE_OID</code><br>
return “base/dboid/[tbackendid_]reloid[_forkname(if not <code>MAIN_FORKNUM</code>)]”<br>
else<br>
return “pg_tblspc/spcoid/PG_(verison)/dboid/[tbackendid_]reloid[_forkname(if not <code>MAIN_FORKNUM</code>)]”</p>
<h2 id="vfd-storagefilefd.c">VFD: storage/file/fd.c</h2>
<h3 id="reason-for-virtual-file-discriptors.">Reason for virtual file discriptors.</h3>
<p>This code manages a cache of ‘virtual’ file descriptors (VFDs).<br>
The server opens many file descriptors for a variety of reasons,<br>
including base tables, scratch files (e.g., sort and hash spool<br>
files), and random calls to C library routines like system(3); it<br>
is quite easy to exceed system limits on the number of open files a<br>
single process can have.  (This is around 256 on many modern<br>
operating systems, but can be as low as 32 on others.)</p>
<h3 id="key-interfaces.">Key interfaces.</h3>
<ul>
<li><code>PathNameOpenFile</code> - used to open virtual files<br>
PathNameOpenFile is intended for files that are held open for a long time, like relation files.</li>
<li><code>OpenTemporaryFile</code> - used to open virtual files<br>
automatically deleted when the File is closed, either explicitly or implicitly at end of transaction or process exit.</li>
</ul>
<p>It is the caller’s responsibility to close them, there is no automatic mechanism in fd.c for that.</p>
<ul>
<li>
<p><code>AllocateFile</code>, <code>AllocateDir</code>, <code>OpenPipeStream</code> and <code>OpenTransientFile</code> are wrappers around <code>fopen(3)</code>, <code>opendir(3)</code>, <code>popen(3)</code> and <code>open(2)</code>, respectively. They behave like the corresponding native functions, except that the handle is registered with the current subtransaction, and will be <em>automatically closed at abort</em>. These are intended mainly for <em>short operations</em> like reading a configuration file; there is a limit on the number of files that can be opened using these functions at any one time.</p>
</li>
<li>
<p><code>BasicOpenFile</code> is just a thin wrapper around open() that can release file descriptors in use by the virtual file descriptors if necessary.</p>
</li>
<li>
<p><code>OpenTransientFile/CloseTransient</code> File for an unbuffered file descriptor</p>
</li>
<li>
<p>VFD structure</p>
</li>
</ul>
<pre class=" language-c"><code class="prism  language-c"><span class="token keyword">typedef</span> <span class="token keyword">struct</span> vfd
<span class="token punctuation">{</span>
	<span class="token keyword">int</span>			fd<span class="token punctuation">;</span>				<span class="token comment">/* current FD, or VFD_CLOSED if none */</span>
	<span class="token keyword">unsigned</span> <span class="token keyword">short</span> fdstate<span class="token punctuation">;</span>		<span class="token comment">/* bitflags for VFD's state */</span>
	ResourceOwner resowner<span class="token punctuation">;</span>		<span class="token comment">/* owner, for automatic cleanup */</span>
	File		nextFree<span class="token punctuation">;</span>		<span class="token comment">/* link to next free VFD, if in freelist */</span>
	File		lruMoreRecently<span class="token punctuation">;</span>	<span class="token comment">/* doubly linked recency-of-use list */</span>
	File		lruLessRecently<span class="token punctuation">;</span>
	off_t		seekPos<span class="token punctuation">;</span>		<span class="token comment">/* current logical file position, or -1 */</span>
	off_t		fileSize<span class="token punctuation">;</span>		<span class="token comment">/* current size of file (0 if not temporary) */</span>
	<span class="token keyword">char</span>	   <span class="token operator">*</span>fileName<span class="token punctuation">;</span>		<span class="token comment">/* name of file, or NULL for unused VFD */</span>
	<span class="token comment">/* NB: fileName is malloc'd, and must be free'd when closing the VFD */</span>
	<span class="token keyword">int</span>			fileFlags<span class="token punctuation">;</span>		<span class="token comment">/* open(2) flags for (re)opening the file */</span>
	<span class="token keyword">int</span>			fileMode<span class="token punctuation">;</span>		<span class="token comment">/* mode to pass to open(2) */</span>
<span class="token punctuation">}</span> Vfd<span class="token punctuation">;</span>

<span class="token comment">/*
 * Virtual File Descriptor array pointer and size.  This grows as
 * needed.  'File' values are indexes into this array.
 * Note that VfdCache[0] is not a usable VFD, just a list header.
 */</span>
<span class="token keyword">static</span> Vfd <span class="token operator">*</span>VfdCache<span class="token punctuation">;</span>
<span class="token keyword">static</span> Size SizeVfdCache <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>

<span class="token comment">/*
 * Number of file descriptors known to be in use by VFD entries.
 */</span>
<span class="token keyword">static</span> <span class="token keyword">int</span>	nfile <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>

<span class="token keyword">typedef</span> <span class="token keyword">int</span> File<span class="token punctuation">;</span> <span class="token comment">/* It indicate the VFD index in VfdCache. */</span>
</code></pre>
<h3 id="key-private-routines-for-vfd-implementation.">Key Private Routines for VFD implementation.</h3>
<p>The <code>VfdCache</code> is a array of VDF. In the cache, there’s a LRU ring and a free list.<br>
The Least Recently Used ring is a doubly linked list(constructs by <code>vfd.lruMoreRecently</code> and <code>vfd.lruLessRecently</code>) that begins and ends on element zero. Element zero is special – it doesn’t represent a file and its “fd” field always == VFD_CLOSED. Element zero is just an anchor that shows us the beginning/end of the ring. Only VFD elements that are currently really open (have an FD assigned) are in the Lru ring. Elements that are “virtually” open can be recognized by having a non-null fileName field.<br>
Free list is a list contains free VDF, constructs by the <code>vfd.nextFree</code>.</p>
<h5 id="initfileaccess-initialize-this-module-during-backend-startup"><code>InitFileAccess</code> initialize this module during backend startup</h5>
<p>For initlization, only init one VFD and set to VfdCache[0],<br>
it’s head of of the VFD cache for both LRU and free list. Set <code>VfdCache-&gt;fd = VFD_CLOSED</code>.<br>
Not <code>SizeVfdCache = 1</code><br>
Also call</p>
<pre class=" language-c"><code class="prism  language-c"><span class="token comment">/* register proc-exit hook to ensure temp files are dropped at exit */</span>
	<span class="token function">on_proc_exit</span><span class="token punctuation">(</span>AtProcExit_Files<span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>to register proc-exit hook to ensure temp files are dropped at exit.(on_proc_exit is a IPC module)<br>
Function <code>AtProcExit_Files</code> is the callback to clean temple files.<br>
It a wraper of <code>CleanupTempFiles(true)</code>, it close temporary files and delete their underlying files.</p>
<h5 id="allocatevfd-grab-a-free-or-new-file-record-from-vfdcache."><code>AllocateVfd</code> grab a free (or new) file record (from VfdCache).</h5>
<p>After <code>InitFileAccess</code>, the <code>SizeVfdCache</code> must bigger than 0 now.<br>
If the free list is empty <code>VfdCache[0].nextFree == 0</code>.<br>
- Double <code>SizeVfdCache</code>, if it’s first time to <code>AllocateVfd</code>. Set <code>SizeVfdCache=32</code>.<br>
- realloc the <code>VfdCache</code> array to siese <code>SizeVfdCache</code>. Note the first element is header.<br>
- Initialize the new entries in <code>VfdCache</code> and link them into the free list.<br>
Get <code>File file</code> from free list head(VfdCache[0].nextFree). The File is actually the id of the VDF in <code>VfdCache</code> array.<br>
Remove the File from free list. <code>VfdCache[0].nextFree = VfdCache[file].nextFree</code>.<br>
Return the allcated vfd file.</p>
<h5 id="freevfd-free-a-file-record."><code>FreeVfd</code> free a file record.</h5>
<p>The input is the VFD File identitor. We can get the actual VFD by it(<code>Vfd *vfdP = &amp;VfdCache[file]</code>).<br>
We should clean the VFD and add it to head of free list.</p>
<pre class=" language-c"><code class="prism  language-c">vfdP<span class="token operator">-&gt;</span>nextFree <span class="token operator">=</span> VfdCache<span class="token punctuation">[</span><span class="token number">0</span><span class="token punctuation">]</span><span class="token punctuation">.</span>nextFree<span class="token punctuation">;</span>
VfdCache<span class="token punctuation">[</span><span class="token number">0</span><span class="token punctuation">]</span><span class="token punctuation">.</span>nextFree <span class="token operator">=</span> file<span class="token punctuation">;</span>
</code></pre>
<h5 id="insert-put-a-file-at-the-front-of-the-lru-ring."><code>Insert</code> put a file at the front of the Lru ring.</h5>
<p>The input is the VFD File identitor. We can get the actual VFD by it(<code>Vfd *vfdP = &amp;VfdCache[file]</code>).<br>
And we insert current file VFD into LRU as the most recentlly used item.<br>
We know the LRU is a ring through <code>lruLessRecently</code> and <code>lruMoreRecently</code> which is alsi <code>File</code> type indicate the index in <code>VfdCache</code>.<br>
If the Vfd item’s <code>VfdCache[file].lruMoreRecently = 0</code> and <code>VfdCache[0].lruLessRecently == file</code>. means the <code>VfdCache[file]</code> is the most recentlly used VFD.</p>
<pre class=" language-c"><code class="prism  language-c">vfdP<span class="token operator">-&gt;</span>lruMoreRecently <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
vfdP<span class="token operator">-&gt;</span>lruLessRecently <span class="token operator">=</span> VfdCache<span class="token punctuation">[</span><span class="token number">0</span><span class="token punctuation">]</span><span class="token punctuation">.</span>lruLessRecently<span class="token punctuation">;</span>
VfdCache<span class="token punctuation">[</span><span class="token number">0</span><span class="token punctuation">]</span><span class="token punctuation">.</span>lruLessRecently <span class="token operator">=</span> file<span class="token punctuation">;</span>
VfdCache<span class="token punctuation">[</span>vfdP<span class="token operator">-&gt;</span>lruLessRecently<span class="token punctuation">]</span><span class="token punctuation">.</span>lruMoreRecently <span class="token operator">=</span> file<span class="token punctuation">;</span>
</code></pre>
<h5 id="delete-delete-a-file-from-the-lru-ring."><code>Delete</code> delete a file from the Lru ring.</h5>
<p>The input is the VFD File identitor. We can get the actual VFD by it(<code>Vfd *vfdP = &amp;VfdCache[file]</code>).<br>
Remove the file VFD from LRU ring.</p>
<pre class=" language-c"><code class="prism  language-c">VfdCache<span class="token punctuation">[</span>vfdP<span class="token operator">-&gt;</span>lruLessRecently<span class="token punctuation">]</span><span class="token punctuation">.</span>lruMoreRecently <span class="token operator">=</span> vfdP<span class="token operator">-&gt;</span>lruMoreRecently<span class="token punctuation">;</span>
VfdCache<span class="token punctuation">[</span>vfdP<span class="token operator">-&gt;</span>lruMoreRecently<span class="token punctuation">]</span><span class="token punctuation">.</span>lruLessRecently <span class="token operator">=</span> vfdP<span class="token operator">-&gt;</span>lruLessRecently<span class="token punctuation">;</span>
</code></pre>
<p>Alyhrough the VFD get remove from LRU, but we may nore free it because we may reuse it later.</p>
<h5 id="lrudelete-remove-a-file-from-the-lru-ring-and-close-its-fd."><code>LruDelete</code> remove a file from the Lru ring and close its FD.</h5>
<p>We can get the actual VFD by it(<code>Vfd *vfdP = &amp;VfdCache[file]</code>).<br>
Make sure the seek position is valid. <code>FilePosIsUnknown(vfdP-&gt;seekPos)</code><br>
Then call <code>close(vfdP-&gt;fd)</code> to close the real FD.<br>
<code>--nfile</code> reduce real used FD count.<br>
And then all <code>Delete</code> to remove vfd from LRU.</p>
<h5 id="releaselrufile-release-an-fd-by-closing-the-last-entry-in-the-lru-ring."><code>ReleaseLruFile</code> Release an fd by closing the last entry in the Lru ring.</h5>
<p>if real used FD count &gt; 0.<br>
Get the less recently used VFD(<code>VfdCache[0].lruMoreRecently</code>) and call <code>LruDelete</code>.<br>
return true.<br>
else return false.</p>
<h5 id="releaselrufiles-release-fds-until-were-under-the-max_safe_fdsmax-fs-allowed-in-the-pg-limit"><code>ReleaseLruFiles</code> Release fd(s) until we’re under the max_safe_fds(max fs allowed in the pg) limit</h5>
<p>If opened fs &gt;= max_safe_fds, run <code>ReleaseLruFile()</code>.</p>
<h5 id="lruinsert-put-a-file-at-the-front-of-the-lru-ring-and-open-it."><code>LruInsert</code> put a file at the front of the Lru ring and open it.</h5>
<p>We can get the actual VFD by it(<code>Vfd *vfdP = &amp;VfdCache[file]</code>).<br>
If the file not opened yet, run <code>ReleaseLruFiles</code> to make sure opened fd count is safe.<br>
Then execute <code>BasicOpenFile</code> to open it. BasicOpenFile, same as open(2) except can free other FDs by <code>ReleaseLruFile</code> if needed.<br>
if open success, <code>++nfile</code>, else return -1.<br>
Then check seek postion is valid, if not <code>--nfile</code> and return -1.<br>
If evething is ok, call <code>Insert(file)</code> to insert it into LRU head.</p>
<h4 id="more-private-routines-for-vfd-implemenation.">More private routines for VFD implemenation.</h4>
<h5 id="fileaccess-open-the-filefile-if-not-opened-add-it-at-the-frount-of-lru-ring."><code>FileAccess</code> open the file(File) if not opened, add it at the frount of LRU ring.</h5>
<p>If the file not opened, call <code>LruInsert</code>.<br>
If the file opened, and the VDF is not head of LRU ring, call <code>Delete(file); Insert(file);</code></p>
<h4 id="vdf-public-interface">VDF public interface</h4>
<h5 id="main-operations">Main operations</h5>
<ul>
<li>
<p>PathNameOpenFile(FileName fileName, int fileFlags, int fileMode): open a file in an arbitrary directory by file name.</p>
<ol>
<li>It’ll can <code>AllocateVfd</code> to retrieve a <code>File file</code>, get it from <code>vfdP = &amp;VfdCache[file]</code>.</li>
<li>Call <code>ReleaseLruFiles</code> to make sure current fd number is safe.</li>
<li>Call <code>BasicOpenFile(fileName, fileFlags, fileMode)</code> to open a real fd.</li>
<li>Call <code>Insert</code> to add it to head of LRU ring.</li>
<li>Set VFD arributes.</li>
</ol>
</li>
<li>
<p>OpenTemporaryFile(bool interXact): Open a temporary file that will disappear when we close it.<br>
It actuall record the <code>File file</code> in current resource owner normally, it’ll fet deleted if not used anymore(transaction end). If we want the temp file outlive the current transaction, we don’t record it.<br>
In either case, the file is removed when the File is explicitly closed.</p>
</li>
<li>
<p>FileClose(File file): close a file when done with it.</p>
<ol>
<li>Get VFD <code>vfdP = &amp;VfdCache[file]</code>, if the file opened, close it by <code>close(cfdP-&gt;fd)</code>, <code>--nfile</code> and mark it closed. Then call <code>Delete(file)</code> to remove file from LRU.</li>
<li>If the file is temporary <code>FD_TEMPORARY</code>, delete it by <code>unlink(vdfP-&gt;fukeName)</code></li>
<li>If it has resource owner, remove it from it’s owner.</li>
<li>call <code>FreeVfd(file)</code> to return vfd to free list.</li>
</ol>
</li>
<li>
<p>FilePrefetch?</p>
</li>
<li>
<p>FileRead(File file, char *buffer, int amount, uint32 wait_event_info)</p>
<ol>
<li>Call <code>FileAccess</code> to a file to make the file as the head of LRU ring.</li>
<li>Call <code>read(vfdP-&gt;fd, buffer, amount)</code> to read data into buffer. if nor success, it’ll retry.</li>
</ol>
</li>
<li>
<p>FileWrite(File file, char *buffer, int amount, uint32 wait_event_info)</p>
<ol>
<li>Call <code>FileAccess</code> to a file to make the file as the head of LRU ring.</li>
<li>If it’s a temporary file, make sure the size not exceeds <code>temp_file_limit</code></li>
<li>Call <code>write(vfdP-&gt;fd, buffer, amount)</code> to write buffer.</li>
<li>Calculate <code>temporary_files_size</code> for temporary file.</li>
<li>if not succeed, may retry.</li>
</ol>
</li>
<li>
<p>FileSync(File file, uint32 wait_event_info): synchronize a file’s in-core state with that on disk</p>
</li>
<li>
<p>FileSeek(File file, off_t offset, int whence): seek read offset.</p>
</li>
<li>
<p>FileTruncate(File file, off_t offset, uint32 wait_event_info): truncate file size.</p>
<ol>
<li><code>FileAccess(file)</code></li>
<li>call <code>ftruncate</code> and update file size in vfd.</li>
</ol>
</li>
<li>
<p>FileWriteback: advise OS that the described dirty data should be flushed</p>
</li>
<li>
<p>FilePathName: get file path name<br>
By return <code>VfdCache[file].fileName</code></p>
</li>
</ul>
<h3 id="fd-wraper-operations-implementations">FD wraper operations Implementations</h3>
<p>They behave like the corresponding native functions, except that the handle is registered with the current subtransaction, and will be <em>automatically closed at abort</em>.</p>
<pre class=" language-c"><code class="prism  language-c"><span class="token comment">/* To specify the file type allocated by these wraper functions */</span>
<span class="token keyword">typedef</span> <span class="token keyword">enum</span>
<span class="token punctuation">{</span>
	AllocateDescFile<span class="token punctuation">,</span>
	AllocateDescPipe<span class="token punctuation">,</span>
	AllocateDescDir<span class="token punctuation">,</span>
	AllocateDescRawFD
<span class="token punctuation">}</span> AllocateDescKind<span class="token punctuation">;</span>

<span class="token comment">/* Each file allocated by these wraper function use AllocateDesc to describe itself. */</span>
<span class="token keyword">typedef</span> <span class="token keyword">struct</span>
<span class="token punctuation">{</span>
	AllocateDescKind kind<span class="token punctuation">;</span>
	SubTransactionId create_subid<span class="token punctuation">;</span>
	<span class="token keyword">union</span>
	<span class="token punctuation">{</span>
		FILE	   <span class="token operator">*</span>file<span class="token punctuation">;</span>
		DIR		   <span class="token operator">*</span>dir<span class="token punctuation">;</span>
		<span class="token keyword">int</span>			fd<span class="token punctuation">;</span>
	<span class="token punctuation">}</span>			desc<span class="token punctuation">;</span>
<span class="token punctuation">}</span> AllocateDesc<span class="token punctuation">;</span>

<span class="token keyword">static</span> <span class="token keyword">int</span>	numAllocatedDescs <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>	<span class="token comment">/* Number of fds opened by these wraper functions */</span>
<span class="token keyword">static</span> <span class="token keyword">int</span>	maxAllocatedDescs <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>  <span class="token comment">/* Max fds than can be opened by these wraper functions */</span>
<span class="token keyword">static</span> AllocateDesc <span class="token operator">*</span>allocatedDescs <span class="token operator">=</span> <span class="token constant">NULL</span><span class="token punctuation">;</span> <span class="token comment">/* Array of the fd' AllocateDescs opened by these wraper functions */</span>
</code></pre>
<h5 id="reserveallocateddescvoid-make-room-for-another-allocateddescs-array-entry-if-needed-and-possible.">reserveAllocatedDesc(void): Make room for another allocatedDescs[] array entry if needed and possible.</h5>
<p>If <code>numAllocatedDescs &lt; maxAllocatedDescs</code>, still have room for new <code>AllocateDescs</code>, return.<br>
If <code>allocatedDescs</code> is null, set <code>maxAllocatedDescs = FD_MINFREE / 2</code> and malloc this size of <code>AllocateDesc</code>.<br>
if <code>allocatedDescs != NULL</code> and no room for new <code>AllocateDescs</code>, set <code>maxAllocatedDescs = max_safe_fds / 2</code> and realloc allocatedDescs to new <code>maxAllocatedDescs</code> size.</p>
<h5 id="ateosubxact_filesbool-iscommit-subtransactionid-mysubid-subtransactionid-parentsubid-take-care-of-subtransaction-commitabort.">AtEOSubXact_Files(bool isCommit, SubTransactionId mySubid, SubTransactionId parentSubid): Take care of subtransaction commit/abort.</h5>
<p>For each element in <code>allocatedDescs</code>, if is’s <code>create_subid == mySubid</code>, for commit, we reassign the files that were opened to the parent subtransaction.  For abort, we close temp files that the subtransaction may have opened.</p>
<h4 id="fd-wraper-operations-interface">FD wraper operations interface</h4>
<h5 id="operations-that-allow-use-of-regular-stdio-----use-with-caution">Operations that allow use of regular stdio — USE WITH CAUTION</h5>
<ul>
<li>AllocateFile: Routines that want to use stdio (ie, FILE*) should use AllocateFile rather than plain fopen()
<ol>
<li><code>reserveAllocatedDesc</code> to make sure <code>AllocateDescs</code> still have room for new opened file.</li>
<li><code>ReleaseLruFiles</code> to close excess fds.</li>
<li><code>fopen</code> to open the file and if success, set <code>AllocateDesc</code> attributes for current <code>allocatedDescs[numAllocatedDescs]</code>.</li>
</ol>
</li>
</ul>
<pre class=" language-c"><code class="prism  language-c">desc<span class="token operator">-&gt;</span>kind <span class="token operator">=</span> AllocateDescFile<span class="token punctuation">;</span>
desc<span class="token operator">-&gt;</span>desc<span class="token punctuation">.</span>file <span class="token operator">=</span> file<span class="token punctuation">;</span>
desc<span class="token operator">-&gt;</span>create_subid <span class="token operator">=</span> <span class="token function">GetCurrentSubTransactionId</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<ul>
<li>FreeFile: Close a file returned by AllocateFile.</li>
</ul>
<h5 id="operations-that-allow-use-of-pipe-streams-popenpclose">Operations that allow use of pipe streams (popen/pclose)</h5>
<ul>
<li>
<p>OpenPipeStream: Routines that want to initiate a pipe stream should use OpenPipeStream rather than plain popen().<br>
Same rountine with <code>AllocateFile</code> but use <code>popen</code>.</p>
</li>
<li>
<p>ClosePipeStream: Close a pipe stream returned by OpenPipeStream</p>
</li>
</ul>
<h5 id="operations-to-allow-use-of-the-dirent.h-library-routines">Operations to allow use of the &lt;dirent.h&gt; library routines</h5>
<ul>
<li>AllocateDir: Routines that want to use &lt;dirent.h&gt; (ie, DIR*) should use AllocateDir rather than plain <code>opendir()</code>.</li>
<li>ReadDir</li>
<li>ReadDirExtended</li>
<li>FreeDir: Close a directory opened with AllocateDir.</li>
</ul>
<h5 id="operations-to-allow-use-of-a-plain-kernel-fd-with-automatic-cleanup">Operations to allow use of a plain kernel FD, with automatic cleanup</h5>
<ul>
<li>OpenTransientFile: Like AllocateFile, but returns an unbuffered fd like <code>open(2)</code></li>
<li>CloseTransientFile: Close a file returned by OpenTransientFile.</li>
</ul>
<h2 id="reinitialization-of-unlogged-relations-storagefilereinit.c">Reinitialization of unlogged relations: storage/file/reinit.c</h2>
<p>During the <code>CREATE TABLE</code> statement, we can specify <code>UNLOGGED</code> in the statement.<br>
If specified, the table is created as an unlogged table. Data written to unlogged tables is not written to the write-ahead log, which makes them considerably faster than ordinary tables. However, they are not crash-safe: an unlogged table is automatically truncated after a crash or unclean shutdown. The contents of an unlogged table are also not replicated to standby servers. Any indexes created on an unlogged table are automatically unlogged as well; however, unlogged GiST indexes are currently not supported and cannot be created on an unlogged table.<br>
So when crash happens. In recovery, unlogged relations may be trashed and must be reset. This should be done BEFORE allowing Hot Standby connections, so that read-only backends don’t try to read whatever garbage is left over from before. <code>ResetUnloggedRelations(UNLOGGED_RELATION_CLEANUP);</code><br>
And then, reset unlogged relations to the contents of their <em>INIT fork</em>. This is done AFTER recovery is complete so as to include any unlogged relations created during recovery, but BEFORE recovery is marked as having completed successfully. <code>ResetUnloggedRelations(UNLOGGED_RELATION_INIT);</code><br>
After <code>CREATE TABLE</code> is called, the table to be created’s <code>relpersistence</code> set to <code>RELPERSISTENCE_UNLOGGED</code>, in table creatation stage(<code>heap_create_with_catalog</code>), <code>heap_create_init_fork</code> get called to create the init fork.</p>
<h4 id="private-routines.">Private routines.</h4>
<h5 id="parse_filename_for_nontemp_relationconst-char-name-int-oidchars-forknumber-fork">parse_filename_for_nontemp_relation(const char *name, int *oidchars, ForkNumber *fork)</h5>
<p>Returns true if the file appears to be in the correct format for a non-temporary relation and false otherwise.<br>
It’ll first parse the oid name and try to match a forkName.<br>
And then try to match segment number. The segment number is for large relations. A single file is too large, it’ll get seperate into segments. The segment name end with “.1”, “.2” an so on.<br>
Temp file path is “base/pgsql_tmp/pg_tblspc(procid).counter” for default tablespace.<br>
All other tablespaces are accessed via symlinks <code>"pg_tblspc/%u/%s/%s", tblspcOid, TABLESPACE_VERSION_DIRECTORY, PG_TEMP_FILES_DIR);</code></p>
<h5 id="resetunloggedrelationsindbspacedirconst-char-dbspacedirname-int-op-process-one-per-dbspace-directory-for-resetunloggedrelations.">ResetUnloggedRelationsInDbspaceDir(const char *dbspacedirname, int op): Process one per-dbspace directory for ResetUnloggedRelations.</h5>
<p>If op includes UNLOGGED_RELATION_CLEANUP, we remove all forks of any relation with an “init” fork, except for the “init” fork itself.<br>
Cleanup is a two-pass operation.  First, we go through and identify all the files with init forks. Then, we go through again and nuke everything with the same OID except the init fork.<br>
For clean up,</p>
<ol>
<li><code>AllocateDir</code> then <code>ReadDir</code>. Iterate the dir, call <code>parse_filename_for_nontemp_relation</code> to filter out none relation file. Insert the file relation entry to a hash table if the relation contains a init fork. If the hash table is none, no need to clean unlogged relations, ie, the unlogged relations must contain a init fork.</li>
<li>If we get the unlogged relations,  <code>AllocateDir</code> then <code>ReadDir</code> iterate it again, we use the hash table to identify unlogged relations, if the fork file is belong to a unlogged relation, <code>unlink</code> it.</li>
</ol>
<p>If op includes UNLOGGED_RELATION_INIT, we copy the “init” fork to the main fork.<br>
Initialization happens after cleanup is complete: we copy each init fork file to the corresponding main fork file.  Note that if we are asked to do both cleanup and init, we may never get here: if the cleanup code determines that there are no init forks in this dbspace, it will return before we get to this point. Cause the hash table build in the clean stage is empty, it’ll return directly.</p>
<ol>
<li><code>AllocateDir</code> then <code>ReadDir</code>. Iterate the dir, if current file is a init fork, call <code>copy_file</code> to copy the init file to a main fork file. <code>copy_file</code> will call <code>pg_flush_data</code> to flush buffer into system kernal buffer.</li>
<li><code>AllocateDir</code> then <code>ReadDir</code>. Iterate the dir, if current file is a init fork, <code>fsync_fname</code> it’s main fork file. <code>fsync</code> flush kernal buffer into disk.</li>
</ol>
<h5 id="resetunloggedrelationsint-op-reset-unlogged-relations-from-before-the-last-restart.-the-interface-of-reset.">ResetUnloggedRelations(int op): Reset unlogged relations from before the last restart. The interface of reset.</h5>
<p>It’ll execute <code>ResetUnloggedRelationsInDbspaceDir</code> for each data directory.</p>
<h2 id="buffile-storagefilebuffile.c">BufFile: storage/file/buffile.c</h2>
<p>Management of large buffered files, primarily temporary files.<br>
BufFile also supports temporary files that exceed the OS file size limit(by opening multiple fd.c temporary files).  This is an essential feature for sorts and hashjoins on large amounts of data.</p>
<h3 id="structures">Structures</h3>
<pre class=" language-c"><code class="prism  language-c"><span class="token comment">/*
 * We break BufFiles into gigabyte-sized segments, regardless of RELSEG_SIZE.
 * The reason is that we'd like large temporary BufFiles to be spread across
 * multiple tablespaces when available.
 */</span>
<span class="token macro property">#<span class="token directive keyword">define</span> MAX_PHYSICAL_FILESIZE	0x40000000</span>
<span class="token macro property">#<span class="token directive keyword">define</span> BUFFILE_SEG_SIZE		(MAX_PHYSICAL_FILESIZE / BLCKSZ)</span>

<span class="token comment">/*
 * This data structure represents a buffered file that consists of one or
 * more physical files (each accessed through a virtual file descriptor
 * managed by fd.c).
 */</span>
<span class="token keyword">struct</span> BufFile
<span class="token punctuation">{</span>
	<span class="token keyword">int</span>			numFiles<span class="token punctuation">;</span>		<span class="token comment">/* number of physical files in set */</span>
	<span class="token comment">/* all files except the last have length exactly MAX_PHYSICAL_FILESIZE */</span>
	File	   <span class="token operator">*</span>files<span class="token punctuation">;</span>			<span class="token comment">/* palloc'd array with numFiles entries */</span>
	off_t	   <span class="token operator">*</span>offsets<span class="token punctuation">;</span>		<span class="token comment">/* palloc'd array with numFiles entries */</span>

	<span class="token comment">/*
	 * offsets[i] is the current seek position of files[i].  We use this to
	 * avoid making redundant FileSeek calls.
	 */</span>

	bool		isTemp<span class="token punctuation">;</span>			<span class="token comment">/* can only add files if this is TRUE */</span>
	bool		isInterXact<span class="token punctuation">;</span>	<span class="token comment">/* keep open over transactions? */</span>
	bool		dirty<span class="token punctuation">;</span>			<span class="token comment">/* does buffer need to be written? */</span>

	<span class="token comment">/*
	 * resowner is the ResourceOwner to use for underlying temp files.  (We
	 * don't need to remember the memory context we're using explicitly,
	 * because after creation we only repalloc our arrays larger.)
	 */</span>
	ResourceOwner resowner<span class="token punctuation">;</span>

	<span class="token comment">/*
	 * "current pos" is position of start of buffer within the logical file.
	 * Position as seen by user of BufFile is (curFile, curOffset + pos).
	 */</span>
	<span class="token keyword">int</span>			curFile<span class="token punctuation">;</span>		<span class="token comment">/* file index (0..n) part of current pos */</span>
	off_t		curOffset<span class="token punctuation">;</span>		<span class="token comment">/* offset part of current pos */</span>
	<span class="token keyword">int</span>			pos<span class="token punctuation">;</span>			<span class="token comment">/* next read/write position in buffer */</span>
	<span class="token keyword">int</span>			nbytes<span class="token punctuation">;</span>			<span class="token comment">/* total # of valid bytes in buffer */</span>
	PGAlignedBlock buffer<span class="token punctuation">;</span>
<span class="token punctuation">}</span><span class="token punctuation">;</span>

<span class="token comment">// The real position for the whole BufFile is equles: curFile * MAX_PHYSICAL_FILESIZE + curOffset + pos</span>
</code></pre>
<h3 id="buffile-intercaces">BufFile intercaces</h3>
<h4 id="buffile-buffilecreatetempbool-interxact-create-a-buffile-for-a-new-temporary-file">BufFile *BufFileCreateTemp(bool interXact): Create a BufFile for a new temporary file</h4>
<p>The file may expand to become multiple temporary files if more than MAX_PHYSICAL_FILESIZE bytes are written to it.</p>
<ol>
<li>Call VFD’s <code>OpenTemporaryFile(interXact);</code> to open a temporary file, get <code>pfile</code>. If interXact is true, the temp file will not be automatically deleted at end of transaction.</li>
<li>Call <code>makeBufFile(pfile)</code> to init the BufFule. It’ll <code>palloc</code> the <code>BufFile</code> in current memory context.
<ul>
<li><code>BufFile *file = (BufFile *) palloc(sizeof(BufFile));</code></li>
<li>Set <code>pfile</code>, <code>file-&gt;files[0] = pfile;</code> and init other attributes.</li>
</ul>
</li>
<li>Set <code>file-&gt;isTemp = true;</code> and <code>file-&gt;isInterXact = interXact;</code></li>
</ol>
<h4 id="buffileclosebuffile-file-close-a-buffile">BufFileClose(BufFile *file): Close a BufFile</h4>
<ol>
<li>Call private routine <code>BufFileFlush(file)</code> tp flush any unwritten data.</li>
<li>Iterate all temporary files in the BufFule, close one by one.</li>
<li>pfree the BufFile object.</li>
</ol>
<h5 id="buffileflushbuffile-file-flush-any-unwritten-data-like-fflush.">BufFileFlush(BufFile *file): flush any unwritten data like fflush.</h5>
<p>If the BufFule is dirty. <code>file-&gt;dirty == true</code>, call private routine <code>BufFileDumpBuffer(file)</code>.<br>
Then if still dirty, return EOF(-1). Else return 0.</p>
<h5 id="buffiledumpbufferbuffile-file-dump-buffer-contents-starting-at-curoffset.">BufFileDumpBuffer(BufFile *file): Dump buffer contents starting at curOffset.</h5>
<p>Start from write position <code>wpos = 0</code>, loop the job.<br>
- Consider add another temp file if <code>file-&gt;curOffset &gt;= MAX_PHYSICAL_FILESIZE &amp;&amp; file-&gt;isTemp</code><br>
- <code>while (file-&gt;curFile + 1 &gt;= file-&gt;numFiles)</code>, call <code>extendBufFile(file)</code><br>
- Increase <code>file-&gt;curFile++</code> and set <code>file-&gt;curOffset = 0L;</code><br>
- Calculate byte to write <code>bytestowrite</code>. If it’s almost end of current temp file, only write <code>MAX_PHYSICAL_FILESIZE - file-&gt;curOffset</code> for current iteration. If not write the whole buffer or the rest of the buffer from last iteration write(which is write to a new extend temp file).<br>
- Make sure the current temp file’s seek position is equals to <code>file-&gt;curOffset</code>.<br>
- Write buffer out by calling vfd’s <code>FileWrite</code>.<br>
- Set current temp file’s seek position by adding wrtite bytes.<br>
- Move current off set <code>file-&gt;curOffset += bytestowrite;</code> and next write position <code>wpos += bytestowrite;</code><br>
Reset <code>file-&gt;curOffset</code> to actual offset position since the loop above use it to write into different temp files.<br>
if <code>file-&gt;curOffset &lt; 0</code> means it shoud pointer to laster file’s offset, so <code>file-&gt;curFile--;</code> and <code>file-&gt;curOffset += MAX_PHYSICAL_FILESIZE;</code>.<br>
No the buffer is empty.</p>
<h5 id="extendbuffilebuffile-file-add-another-component-temp-file.">extendBufFile(BufFile *file): Add another component temp file.</h5>
<ol>
<li>Call VFD’s <code>OpenTemporaryFile(file-&gt;isInterXact);</code> to open a new temp file.</li>
<li><code>repalloc</code> the <code>file-&gt;files</code> amd <code>file-&gt;offsets</code> by add one more room.</li>
<li>Set correspond <code>file-&gt;files[file-&gt;numFiles] = pfile;</code> and <code>file-&gt;offsets[file-&gt;numFiles] = 0L;</code>. Increase <code>file-&gt;numFiles++</code>.</li>
</ol>
<h4 id="size_t-buffilereadbuffile-file-void-ptr-size_t-size-like-fread-except-we-assume-1-byte-element-size.">size_t BufFileRead(BufFile *file, void *ptr, size_t size): Like fread() except we assume 1-byte element size.</h4>
<ol>
<li>If file is dirty, flush it by <code>BufFileFlush(file)</code></li>
<li>loop until we read the required <code>size</code>.
<ul>
<li>If the fetch start pos larger than total bytes: <code>file-&gt;pos &gt;= file-&gt;nbytes</code>. Set <code>file-&gt;curOffset += file-&gt;pos;</code>, and clear the buffer <code>file-&gt;pos = 0; file-&gt;nbytes = 0;</code>. Call <code>BufFileLoadBuffer(file);</code> to load buffer from <code>file-&gt;curOffset</code> position.</li>
<li>Calculate current iteration read size <code>nthistime</code> base on current buffer. If required size larger than current unread buffer size, only read current buffer size and set next required size to <code>size -= nthistime;</code>. If required size is less than current unread buffer size, read the size.</li>
</ul>
</li>
<li>return total read size.</li>
</ol>
<h5 id="buffileloadbufferbuffile-file-load-some-data-into-buffer-if-possible-starting-from-curoffset.">BufFileLoadBuffer(BufFile *file): Load some data into buffer, if possible, starting from curOffset.</h5>
<ol>
<li>If <code>file-&gt;curOffset &gt;= MAX_PHYSICAL_FILESIZE &amp;&amp; file-&gt;curFile + 1 &lt; file-&gt;numFiles</code>, we need increase <code>file-&gt;curFile++;</code> and <code>set file-&gt;curOffset = 0L;</code> means read next temp file.</li>
<li>Make sure the current temp file’s seek position is equals to <code>file-&gt;curOffset</code>.</li>
<li>Read next buffer for current temp file. And set <code>file-&gt;nbytes</code> to the read bytes number.</li>
</ol>
<h4 id="size_t-buffilewritebuffile-file-void-ptr-size_t-size-like-fwrite-except-we-assume-1-byte-element-size.">size_t BufFileWrite(BufFile *file, void *ptr, size_t size): Like fwrite() except we assume 1-byte element size.</h4>
<p>Loop until the required write size is &lt; 1.</p>
<ol>
<li>if curent buffer position is larger or equal than block size <code>file-&gt;pos &gt;= BLCKSZ</code>. Buffer is full, dump it out by calling <code>BufFileDumpBuffer(file)</code> if it’s dirty.<br>
Or, it may comes from read operation that fill the buffer full. Increase the current offset <code>file-&gt;curOffset += file-&gt;pos;</code> and set <code>file-&gt;pos = 0; file-&gt;nbytes = 0;</code></li>
<li>Calculate the max buffer size we can write for current iteration. <code>nthistime = BLCKSZ - file-&gt;pos;</code><br>
if required write size &gt; <code>nthistime</code>, write current buffer size for current iteration, else write required size.</li>
<li>Update <code>file-&gt;dirty = true; file-&gt;pos += nthistime; ...</code> reset the required write size for next iteration.<br>
Return writed size.</li>
</ol>
<h4 id="buffileseekbuffile-file-int-fileno-off_t-offset-int-whence-like-fseek">BufFileSeek(BufFile *file, int fileno, off_t offset, int whence): Like fseek()</h4>
<p>Target position needs two values in order to work when logical filesize exceeds maximum value representable by long.<br>
Only implement SEEK_SET and SEEK_CUR.<br>
The main idea is to calculate <code>file-&gt;curFile</code>, <code>file-&gt;curOffset</code>, and <code>file-&gt;pos</code>.<br>
If seek is to a point within existing buffer; we can just adjust pos-within-buffer, without flushing buffer.<br>
Otherwise, must reposition buffer, so flush any dirty data. Flush may create new temp file.<br>
Calculate file number if needed.</p>
<h4 id="void-buffiletellbuffile-file-int-fileno-off_t-offset-retrieve-current-file-number-and-seek-offset">void BufFileTell(BufFile *file, int *fileno, off_t *offset): Retrieve current file number and seek offset</h4>
<h4 id="buffileseekblockbuffile-file-long-blknum-block-oriented-seek">BufFileSeekBlock(BufFile *file, long blknum): block-oriented seek</h4>
<p>Calculate fileno and offset by blknum, call <code>BufFileSeek</code>.</p>

