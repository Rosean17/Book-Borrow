

<script>
  import { onMount } from 'svelte';
  import { supabase } from './supabase.js';

  let activeTab = 'Books';
  let search = '';
  let darkMode = false;

 /** @type {any[]} */
let books = [];

/** @type {any[]} */
let borrowers = [];
  let loading = false;
  let saving = false;
  let errorMessage = '';
  let successMessage = '';
  let editingBook = null;
let editingBorrower = null;

let editBookForm = {
  title: '',
  author: '',
  isbn: '',
  category: '',
  total_copies: 1
};

let editBorrowerForm = {
  full_name: '',
  student_id: '',
  contact: ''
};

  /** @type {any[]} */
  let borrowRecords = [];
  let selectedBookId = '';
  let selectedBorrowerId = '';
  let borrowedAt = new Date().toISOString().slice(0, 10);
  let dueDate = new Date(Date.now() + 14 * 86400000).toISOString().slice(0, 10);

  $: filteredBooks = books.filter((book) =>
    `${book.title} ${book.author} ${book.category || ''} ${book.isbn || ''}`
      .toLowerCase()
      .includes(search.toLowerCase())
  );

  $: filteredBorrowers = borrowers.filter((borrower) =>
    `${borrower.full_name} ${borrower.student_id} ${borrower.contact || ''}`
      .toLowerCase()
      .includes(search.toLowerCase())
  );

  $: totalCopies = books.reduce(
    (sum, book) => sum + Number(book.total_copies || 0),
    0
  );

  $: availableCopies = books.reduce(
    (sum, book) => sum + Number(book.available_copies || 0),
    0
  );

  $: borrowedCopies = totalCopies - availableCopies;

  onMount(() => {
    loadBooks();
    loadBorrowers();
    loadBorrowRecords();
  });

  async function loadBooks() {
    loading = true;
    errorMessage = '';

    const { data, error } = await supabase
      .from('books')
      .select('*')
      .order('created_at', { ascending: false });

    if (error) {
      errorMessage = `Could not load books: ${error.message}`;
    } else {
      books = data ?? [];
    }

    loading = false;
  }

  async function loadBorrowers() {
    const { data, error } = await supabase
      .from('borrowers')
      .select('*')
      .order('created_at', { ascending: false });

    if (error) {
      errorMessage = `Could not load borrowers: ${error.message}`;
    } else {
      borrowers = data ?? [];
    }
  }

  async function addBook() {
    const title = prompt('Enter book title:');
    if (!title?.trim()) return;

    const author = prompt('Enter author:');
    if (!author?.trim()) return;

    const isbn = prompt('Enter ISBN (optional):', '');
    if (isbn === null) return;

    const category = prompt('Enter category:', 'General');
    if (category === null) return;

    const totalInput = prompt('Enter total copies:', '1');
    if (totalInput === null) return;

    const total = Number(totalInput);

    if (!Number.isInteger(total) || total < 1) {
      alert('Enter a whole number of copies greater than zero.');
      return;
    }

    saving = true;
    errorMessage = '';
    successMessage = '';

    const { error } = await supabase.from('books').insert({
      title: title.trim(),
      author: author.trim(),
      isbn: isbn.trim() || null,
      category: category.trim() || 'General',
      total_copies: total,
      available_copies: total
    });

    saving = false;

    if (error) {
      errorMessage = `Could not add book: ${error.message}`;
      return;
    }

    successMessage = 'Book added successfully.';
    await loadBooks();
  }

  /** @param {any} book */
  async function editBook(book) {
    const title = prompt('Update book title:', book.title);
    if (!title?.trim()) return;

    const author = prompt('Update author:', book.author);
    if (!author?.trim()) return;

    const isbn = prompt('Update ISBN:', book.isbn || '');
    if (isbn === null) return;

    const category = prompt('Update category:', book.category || 'General');
    if (category === null) return;

    const borrowed = Number(book.total_copies) -
      Number(book.available_copies);

    const totalInput = prompt(
      `Update total copies (minimum ${borrowed}):`,
      String(book.total_copies)
    );

    if (totalInput === null) return;

    const total = Number(totalInput);

    if (!Number.isInteger(total) || total < borrowed || total < 0) {
      alert(`Total copies cannot be less than ${borrowed} borrowed copies.`);
      return;
    }

    saving = true;
    errorMessage = '';
    successMessage = '';

    const { error } = await supabase
      .from('books')
      .update({
        title: title.trim(),
        author: author.trim(),
        isbn: isbn.trim() || null,
        category: category.trim() || 'General',
        total_copies: total,
        available_copies: total - borrowed
      })
      .eq('id', book.id);

    saving = false;

    if (error) {
      errorMessage = `Could not update book: ${error.message}`;
      return;
    }

    successMessage = 'Book updated successfully.';
    await loadBooks();
  }

  /** @param {string} id */
  async function deleteBook(id) {
    if (!confirm('Delete this book?')) return;

    saving = true;
    errorMessage = '';
    successMessage = '';

    const { error } = await supabase
      .from('books')
      .delete()
      .eq('id', id);

    saving = false;

    if (error) {
      errorMessage = `Could not delete book: ${error.message}`;
      return;
    }

    successMessage = 'Book deleted successfully.';
    await loadBooks();
  }

  async function addBorrower() {
    const fullName = prompt('Enter borrower full name:');
    if (!fullName?.trim()) return;

    const studentId = prompt('Enter student ID:');
    if (!studentId?.trim()) return;

    const contact = prompt('Enter contact number (optional):', '');
    if (contact === null) return;

    saving = true;
    errorMessage = '';
    successMessage = '';

    const { error } = await supabase.from('borrowers').insert({
      full_name: fullName.trim(),
      student_id: studentId.trim(),
      contact: contact.trim() || null
    });

    saving = false;

    if (error) {
      errorMessage = `Could not add borrower: ${error.message}`;
      return;
    }

    successMessage = 'Borrower added successfully.';
    await loadBorrowers();
  }

  /** @param {any} borrower */
  async function editBorrower(borrower) {
    const fullName = prompt('Update borrower name:', borrower.full_name);
    if (!fullName?.trim()) return;

    const studentId = prompt('Update student ID:', borrower.student_id);
    if (!studentId?.trim()) return;

    const contact = prompt('Update contact:', borrower.contact || '');
    if (contact === null) return;

    saving = true;
    errorMessage = '';
    successMessage = '';

    const { error } = await supabase
      .from('borrowers')
      .update({
        full_name: fullName.trim(),
        student_id: studentId.trim(),
        contact: contact.trim() || null
      })
      .eq('id', borrower.id);

    saving = false;

    if (error) {
      errorMessage = `Could not update borrower: ${error.message}`;
      return;
    }

    successMessage = 'Borrower updated successfully.';
    await loadBorrowers();
  }

  /** @param {string} id */
  async function deleteBorrower(id) {
    if (!confirm('Delete this borrower?')) return;

    saving = true;
    errorMessage = '';
    successMessage = '';

    const { error } = await supabase
      .from('borrowers')
      .delete()
      .eq('id', id);

    saving = false;

    if (error) {
      errorMessage = `Could not delete borrower: ${error.message}`;
      return;
    }

    successMessage = 'Borrower deleted successfully.';
    await loadBorrowers();
  }


  async function loadBorrowRecords() {
    const { data, error } = await supabase
      .from('borrow_records')
      .select('*')
      .order('created_at', { ascending: false });
    if (error) {
      errorMessage = `Could not load borrowing records: ${error.message}`;
      return;
    }
    borrowRecords = data ?? [];
  }

  async function borrowBook() {
    if (!selectedBorrowerId || !selectedBookId) {
      errorMessage = 'Please select both a student and a book.';
      return;
    }
    if (!borrowedAt || !dueDate || dueDate < borrowedAt) {
      errorMessage = 'Please choose valid borrowing and due dates.';
      return;
    }
    saving = true;
    errorMessage = '';
    successMessage = '';
    const { error } = await supabase.rpc('borrow_book', {
      p_book_id: selectedBookId,
      p_borrower_id: selectedBorrowerId,
      p_borrowed_at: borrowedAt,
      p_due_date: dueDate
    });
    saving = false;
    if (error) {
      errorMessage = `Could not borrow book: ${error.message}`;
      return;
    }
    successMessage = 'Borrowing saved. Available copies have been reduced by one.';
    await Promise.all([loadBooks(), loadBorrowRecords()]);
  }

  /** @param {any} record */
  async function editBorrowRecord(record) {
    if (record.returned_at) {
      alert('A returned record cannot be edited.');
      return;
    }
    const bookOptions = books.map((b, i) => `${i + 1}. ${b.title} (available: ${b.available_copies})`).join('\\n');
    const bookChoice = prompt(`Choose book number:\\n${bookOptions}`, String(Math.max(1, books.findIndex((b) => b.id === record.book_id) + 1)));
    if (bookChoice === null) return;
    const bookIndex = Number(bookChoice) - 1;
    if (!Number.isInteger(bookIndex) || bookIndex < 0 || bookIndex >= books.length) {
      alert('Please enter a valid book number.');
      return;
    }
    const borrowerOptions = borrowers.map((b, i) => `${i + 1}. ${b.full_name} (${b.student_id})`).join('\\n');
    const borrowerChoice = prompt(`Choose student number:\\n${borrowerOptions}`, String(Math.max(1, borrowers.findIndex((b) => b.id === record.borrower_id) + 1)));
    if (borrowerChoice === null) return;
    const borrowerIndex = Number(borrowerChoice) - 1;
    if (!Number.isInteger(borrowerIndex) || borrowerIndex < 0 || borrowerIndex >= borrowers.length) {
      alert('Please enter a valid student number.');
      return;
    }
    const nextBorrowedAt = prompt('Borrowed date (YYYY-MM-DD):', record.borrowed_at);
    if (!nextBorrowedAt) return;
    const nextDueDate = prompt('Due date (YYYY-MM-DD):', record.due_date);
    if (!nextDueDate) return;
    if (nextDueDate < nextBorrowedAt) {
      alert('Due date cannot be earlier than the borrowed date.');
      return;
    }
    saving = true;
    errorMessage = '';
    successMessage = '';
    const { error } = await supabase.rpc('edit_borrow_record', {
      p_record_id: record.id,
      p_book_id: books[bookIndex].id,
      p_borrower_id: borrowers[borrowerIndex].id,
      p_borrowed_at: nextBorrowedAt,
      p_due_date: nextDueDate
    });
    saving = false;
    if (error) {
      errorMessage = `Could not edit borrowing record: ${error.message}`;
      return;
    }
    successMessage = 'Borrowing record updated.';
    await Promise.all([loadBooks(), loadBorrowRecords()]);
  }

  /** @param {any} record */
  async function returnBook(record) {
    if (record.returned_at) {
      alert('This book has already been returned.');
      return;
    }
    if (!confirm('Mark this book as returned? One available copy will be restored.')) return;
    saving = true;
    errorMessage = '';
    successMessage = '';
    const { error } = await supabase.rpc('return_book', { p_record_id: record.id });
    saving = false;
    if (error) {
      errorMessage = `Could not return book: ${error.message}`;
      return;
    }
    successMessage = 'Book returned. One available copy has been restored.';
    await Promise.all([loadBooks(), loadBorrowRecords()]);
  }

  /** @param {string} tab */
  function switchTab(tab) {
    activeTab = tab;
    search = '';
    errorMessage = '';
    successMessage = '';

    if (tab === 'Books') loadBooks();
    if (tab === 'Borrowers') loadBorrowers();
    if (tab === 'Borrow & Return') { loadBorrowRecords(); loadBooks(); loadBorrowers(); }
  }
</script>


<svelte:head>
  <title>BookWise | Book Borrowing System</title>
  <meta
    name="description"
    content="BookWise library book and borrower management system"
  />
</svelte:head>

<div class="app" class:dark={darkMode}>
  <aside class="sidebar">
    <div class="brand">
      <div class="brand-icon">B</div>
      <div>
        <h2>BookWise</h2>
        <span>Library Management</span>
      </div>
    </div>

    <p class="nav-label">MAIN MENU</p>

    {#each ['Dashboard', 'Books', 'Borrowers', 'Borrow & Return'] as tab}
      <button
        class="nav-item"
        class:active={activeTab === tab}
        onclick={() => switchTab(tab)}
      >
        <span class="nav-icon">
          {tab === 'Dashboard' ? '▦' :
            tab === 'Books' ? '▤' :
            tab === 'Borrowers' ? '♙' : '↔'}
        </span>
        {tab}
      </button>
    {/each}

    <div class="sidebar-footer">
      <div class="avatar">LB</div>
      <div>
        <strong>Library Admin</strong>
        <span>Administrator</span>
      </div>
    </div>
  </aside>

  <main class="main">
    <header class="topbar">
      <div>
        <span class="eyebrow">LIBRARY MANAGEMENT</span>
        <h1>{activeTab}</h1>
      </div>

      <div class="top-right">
        <button
          class="theme-toggle"
          onclick={() => darkMode = !darkMode}
          aria-label={darkMode ? 'Switch to light mode' : 'Switch to dark mode'}
          title={darkMode ? 'Switch to light mode' : 'Switch to dark mode'}
        >
          {#if darkMode}
            <span>☀️</span> Light Mode
          {:else}
            <span>🌙</span> Dark Mode
          {/if}
        </button>
        <span class="online-dot"></span>
        <span>System Preview</span>
      </div>
    </header>

    {#if errorMessage}
      <div class="app-message error-message" role="alert">{errorMessage}</div>
    {/if}
    {#if successMessage}
      <div class="app-message success-message" role="status">{successMessage}</div>
    {/if}

    {#if activeTab === 'Dashboard'}
      <section class="welcome">
        <span class="eyebrow">OVERVIEW</span>
        <h2>Welcome to BookWise</h2>
        <p>Manage your books, borrowers, and library borrowing activities.</p>
      </section>

      <section class="stats">
        <article class="stat-card">
          <span>Total Book Titles</span>
          <strong>{books.length}</strong>
          <small>Registered titles</small>
        </article>

        <article class="stat-card">
          <span>Total Copies</span>
          <strong>{totalCopies}</strong>
          <small>All inventory</small>
        </article>

        <article class="stat-card">
          <span>Available Copies</span>
          <strong>{availableCopies}</strong>
          <small>Ready to borrow</small>
        </article>

        <article class="stat-card">
          <span>Borrowed Copies</span>
          <strong>{borrowedCopies}</strong>
          <small>Currently unavailable</small>
        </article>

        <article class="stat-card">
          <span>Registered Borrowers</span>
          <strong>{borrowers.length}</strong>
          <small>Registered people</small>
        </article>
      </section>

      <section class="panel">
        <div class="panel-heading">
          <div>
            <h3>Book Catalog</h3>
            <p class="muted">A quick look at your registered books.</p>
          </div>

          <button class="primary-btn" onclick={() => switchTab('Books')}>
            View All Books →
          </button>
        </div>

        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>BOOK TITLE</th>
                <th>AUTHOR</th>
                <th>CATEGORY</th>
                <th>AVAILABLE</th>
              </tr>
            </thead>
            <tbody>
              {#each books.slice(0, 5) as book (book.id)}
                <tr>
                  <td><strong>{book.title}</strong></td>
                  <td>{book.author}</td>
                  <td><span class="tag">{book.category}</span></td>
                  <td>{book.available_copies} / {book.total_copies}</td>
                </tr>
              {:else}
                <tr><td colspan="4" class="empty">No books registered yet.</td></tr>
              {/each}
            </tbody>
          </table>
        </div>
      </section>

    {:else if activeTab === 'Books'}
      <section class="stats">
        <article class="stat-card">
          <span>Total Titles</span>
          <strong>{books.length}</strong>
          <small>Book records</small>
        </article>

        <article class="stat-card">
          <span>Total Copies</span>
          <strong>{totalCopies}</strong>
          <small>All inventory</small>
        </article>

        <article class="stat-card">
          <span>Available Copies</span>
          <strong>{availableCopies}</strong>
          <small>Copies in library</small>
        </article>
      </section>

      <section class="panel">
        <div class="panel-heading">
          <div>
            <h3>Book Catalog</h3>
            <p class="muted">Manage and organize your library collection.</p>
          </div>
          <button class="primary-btn" onclick={addBook}>＋ Add Book</button>
        </div>

        <input
          class="search"
          bind:value={search}
          placeholder="⌕  Search title, author, or category..."
          aria-label="Search books"
        />

        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>BOOK DETAILS</th>
                <th>CATEGORY</th>
                <th>TOTAL</th>
                <th>AVAILABLE</th>
                <th>ACTIONS</th>
              </tr>
            </thead>
            <tbody>
              {#each filteredBooks as book (book.id)}
                <tr>
                  <td>
                    <div class="book-title">{book.title}</div>
                    <div class="muted small">{book.author}</div>
                  </td>
                  <td><span class="tag">{book.category}</span></td>
                  <td>{book.total_copies}</td>
                  <td>
                    <span class="availability">
                      {book.available_copies} available
                    </span>
                  </td>
                  <td class="actions">
                    <button class="icon-btn" title="Edit book" aria-label="Edit book" onclick={() => editBook(book)}>✎</button>
                    <button class="icon-btn danger" title="Delete book" aria-label="Delete book" onclick={() => deleteBook(book.id)}>×</button>
                  </td>
                </tr>
              {:else}
                <tr><td colspan="5" class="empty">No books found.</td></tr>
              {/each}
            </tbody>
          </table>
        </div>

        <p class="table-foot">
          Showing {filteredBooks.length} of {books.length} books
        </p>
      </section>

    {:else if activeTab === 'Borrowers'}
      <section class="stats">
        <article class="stat-card">
          <span>Registered Borrowers</span>
          <strong>{borrowers.length}</strong>
          <small>All borrower records</small>
        </article>

        <article class="stat-card">
          <span>Active Borrowers</span>
          <strong>{borrowers.length}</strong>
          <small>Registered and active</small>
        </article>

        <article class="stat-card">
          <span>Borrowed Copies</span>
          <strong>{borrowedCopies}</strong>
          <small>Inventory currently out</small>
        </article>
      </section>

      <section class="panel">
        <div class="panel-heading">
          <div>
            <h3>Borrower Directory</h3>
            <p class="muted">Manage registered library borrowers.</p>
          </div>
          <button class="primary-btn" onclick={addBorrower}>＋ Add Borrower</button>
        </div>

        <input
          class="search"
          bind:value={search}
          placeholder="⌕  Search name or student ID..."
          aria-label="Search borrowers"
        />

        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>BORROWER</th>
                <th>STUDENT ID</th>
                <th>CONTACT</th>
                <th>ACTIVE LOANS</th>
                <th>ACTIONS</th>
              </tr>
            </thead>
            <tbody>
              {#each filteredBorrowers as borrower (borrower.id)}
                <tr>
                  <td><strong>{borrower.full_name}</strong></td>
                  <td>{borrower.student_id}</td>
                  <td>{borrower.contact || '—'}</td>
                  <td>{borrowRecords.filter((r) => r.borrower_id === borrower.id && !r.returned_at).length}</td>
                  <td class="actions">
                    <button class="icon-btn" title="Edit borrower" aria-label="Edit borrower" onclick={() => editBorrower(borrower)}>✎</button>
                    <button class="icon-btn danger" title="Delete borrower" aria-label="Delete borrower" onclick={() => deleteBorrower(borrower.id)}>×</button>
                  </td>
                </tr>
              {:else}
                <tr><td colspan="5" class="empty">No borrowers found.</td></tr>
              {/each}
            </tbody>
          </table>
        </div>

        <p class="table-foot">
          Showing {filteredBorrowers.length} of {borrowers.length} borrowers
        </p>
      </section>

    {:else}
      <section class="stats">
        <article class="stat-card">
          <span>Available Copies</span>
          <strong>{availableCopies}</strong>
          <small>Ready to borrow</small>
        </article>

        <article class="stat-card">
          <span>Borrowed Copies</span>
          <strong>{borrowedCopies}</strong>
          <small>Currently unavailable</small>
        </article>

        <article class="stat-card">
          <span>Registered Borrowers</span>
          <strong>{borrowers.length}</strong>
          <small>Borrower records</small>
        </article>
      </section>

      <section class="panel">
        <div class="panel-heading">
          <div>
            <h3>Borrow a Book</h3>
            <p class="muted">Select a registered student and an available book. Saving the loan deducts one available copy.</p>
          </div>
        </div>
        <div class="borrow-form">
          <label>
            Student
            <select bind:value={selectedBorrowerId} disabled={saving}>
              <option value="">Choose a student</option>
              {#each borrowers as borrower (borrower.id)}
                <option value={borrower.id}>{borrower.full_name} — {borrower.student_id}</option>
              {/each}
            </select>
          </label>
          <label>
            Book
            <select bind:value={selectedBookId} disabled={saving}>
              <option value="">Choose an available book</option>
              {#each books.filter((book) => Number(book.available_copies) > 0) as book (book.id)}
                <option value={book.id}>{book.title} — {book.available_copies} available</option>
              {/each}
            </select>
          </label>
          <label>
            Date borrowed
            <input type="date" bind:value={borrowedAt} disabled={saving} />
          </label>
          <label>
            Due date
            <input type="date" bind:value={dueDate} disabled={saving} />
          </label>
          <button class="primary-btn" onclick={borrowBook} disabled={saving || !books.length || !borrowers.length}>
            {saving ? 'Saving…' : '＋ Confirm Borrowing'}
          </button>
        </div>
      </section>

      <section class="panel">
        <div class="panel-heading">
          <div>
            <h3>Borrowing Records</h3>
            <p class="muted">Every active loan and returned book is saved here.</p>
          </div>
          <button class="secondary-btn" onclick={() => { loadBorrowRecords(); loadBooks(); loadBorrowers(); }}>↻ Refresh</button>
        </div>
        <input class="search" bind:value={search} placeholder="Search student or book title..." aria-label="Search borrowing records" />
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>STUDENT</th>
                <th>BOOK</th>
                <th>DATE BORROWED</th>
                <th>DUE DATE</th>
                <th>STATUS</th>
                <th>ACTIONS</th>
              </tr>
            </thead>
            <tbody>
              {#each borrowRecords.filter((r) => {
                const borrower = borrowers.find((b) => b.id === r.borrower_id);
                const book = books.find((b) => b.id === r.book_id);
                return `${borrower?.full_name || ''} ${borrower?.student_id || ''} ${book?.title || ''}`.toLowerCase().includes(search.toLowerCase());
              }) as record (record.id)}
                {@const recordBorrower = borrowers.find((b) => b.id === record.borrower_id)}
                {@const recordBook = books.find((b) => b.id === record.book_id)}
                <tr>
                  <td><strong>{recordBorrower?.full_name || 'Unknown student'}</strong><div class="muted small">{recordBorrower?.student_id || ''}</div></td>
                  <td>{recordBook?.title || 'Unknown book'}</td>
                  <td>{record.borrowed_at}</td>
                  <td>{record.due_date}</td>
                  <td><span class="status">{record.returned_at ? 'Returned' : (record.due_date < new Date().toISOString().slice(0, 10) ? 'Overdue' : 'Borrowed')}</span></td>
                  <td class="actions">
                    {#if !record.returned_at}
                      <button class="icon-btn" title="Edit borrowing record" aria-label="Edit borrowing record" onclick={() => editBorrowRecord(record)}>✎</button>
                      <button class="primary-btn" onclick={() => returnBook(record)} disabled={saving}>Return</button>
                    {:else}
                      <span class="muted small">Completed</span>
                    {/if}
                  </td>
                </tr>
              {:else}
                <tr><td colspan="6" class="empty">No borrowing records found.</td></tr>
              {/each}
            </tbody>
          </table>
        </div>
      </section>
    {/if}
  </main>
</div>
