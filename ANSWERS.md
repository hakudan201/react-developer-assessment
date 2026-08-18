# Assessment answers

## Section 1 — Read and Fix (Backend)

### Question 1.1 — Find the Bugs

This endpoint should create a new note and return it. Find bugs, explain what goes wrong from the user's perspective, and write the corrected code.

```javascript
app.post('/api/notes', (req, res) => {
  try {
    const note = Note.create({
      title: req.body.title,
      content: req.body.content,
    });
    res.status(200).json(note);
  } catch (error) {
    console.log(error);
  }
});
```

**Answer format (for each bug):**

- The line with the problem
- What the user experiences because of it
- The fix

1:
- Line / problem: Catch only log error and never return response.
- What the user experiences: When error, the client receives no response and then timeout.
- The fix:
```javascript
return res.status(500).json({message: "Note creation error"});
```

2:
- Line / problem: Asynchronous operation doesnt use async/await.
- What the user experiences: The client may receive an empty response.
- The fix:
```javascript
app.post('/api/notes', async(req, res) => {...})
const note = await Note.create({...});
```

3:
- Line / problem: Standard success response code.
- What the user experiences: No obvious effect for the user, but the API returns an inaccurate HTTP status code.
- The fix:
```javascript
res.status(201).json(note);
```

4:
- Line / problem: title and content are passed into Note.create without check.
- What the user experiences: Saving with a missing or empty title/content can still create a junk note.
- The fix: If title or content is missing/empty, return Bad Request code and skip create.

```javascript
const { title, content } = req.body;
if (!title || !content) {
  return res.status(400).json({ message: 'Title and content are required' });
}
```

---

### Question 1.2 — What's Wrong With This Schema?

```javascript
const userSchema = new mongoose.Schema({
  name: String,
  email: String,
  age: String,
  createdAt: String,
});
```

List everything you would change and explain **why** for each change. There is no single right answer — we want to see your reasoning and what you prioritize.

- **age:** Use `Number`. Because age is numeric data and may need comparisons or calculations.
- **createdAt:** use `Date`, because it represents a date/time value. Also give it a default value `Date.now`.
- **email:** can make it `unique`(depends on system requirement). If `unique`, make it `required`. Also make them lowercase so a@gmail.com and A@GmAiL.com wont be 2 different emails.
- **name:** make it `required` because every user should have a name.

---

### Question 1.3 — This Query Is Slow

The notes collection has 500,000 documents. This endpoint takes 8 seconds to respond:

```javascript
app.get('/api/notes/search', async (req, res) => {
  const notes = await Note.find({});
  const results = notes.filter((n) => n.title.toLowerCase().includes(req.query.q.toLowerCase()));
  res.json(results);
});
```

1. Explain **why** it is slow.
2. Rewrite it to be faster. You don't need perfect syntax — show the approach.

Why it is slow: It get everything and then filter in JS(not DB).

```javascript
app.get('/api/notes/search', async (req, res) => {
  const q = req.query.q;
  if (!q) {
    return res.json([]);
  }

  const results = await Note.find({
    title: { $regex: q, $options: 'i' },
  });

  res.json(results);
});
```

---

## Section 2 — Read and Fix (Frontend)

### Question 2.1 — Fix This React Component

This component should show a list of notes and let the user delete one. Find bugs and fix each one.

```jsx
function NoteList() {
  const [notes, setNotes] = useState();

  useEffect(async () => {
    const res = await fetch('/api/notes');
    const data = await res.json();
    setNotes(data);
  }, []);

  function handleDelete(id) {
    fetch(`/api/notes/${id}`, { method: 'DELETE' });
    setNotes(notes.filter((n) => n.id !== id));
  }

  return (
    <ul>
      {notes.map((note) => (
        <li>
          {note.title}
          <button onClick={handleDelete(note._id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

**Your answer format (for each problem):**

- Quote the line with the problem
- Explain what goes wrong
- Write the fix

1. The line with the problem: `const [notes, setNotes] = useState();`
- What goes wrong: `notes` is `undefined`. First render runs `notes.map` and the page crashes.
- The fix: `useState([])` so map always has an array.

2. The line with the problem: `useEffect(async () => {`
- What goes wrong: the effect function cannot be `async`. React expects it to return nothing or a cleanup function, not a Promise.
- The fix: keep the effect sync, put `async` in an inner function.

```jsx
useEffect(() => {
  async function load() {
    const res = await fetch('/api/notes');
    const data = await res.json();
    setNotes(data);
  }
  load();
}, []);
```

3. The line with the problem: `onClick={handleDelete(note._id)}`
- What goes wrong: `handleDelete` runs on every render, not on click. Delete fires immediately; `onClick` gets `undefined`.
- The fix: `onClick={() => handleDelete(note._id)}`

4. The line with the problem: `notes.filter((n) => n.id !== id)` vs `note._id`
- What goes wrong: the button passes `_id`, the filter looks at `id`. Nothing is removed from the list.
- The fix: use the same field, `n._id !== id`.

5. The line with the problem: Missing `key` on `<li>`
- What goes wrong: React cannot track list items. After delete, the UI can mix up rows.
- The fix: `<li key={note._id}>`

6. `fetch(...)` then `setNotes` with no wait
- What goes wrong: the UI removes the note even if the server delete fails. Also `setNotes(notes.filter(...))` can use a stale `notes`.
- The fix: `await` the delete (and check `res.ok`), then update with `setNotes((prev) => prev.filter(...))`.

```jsx
async function handleDelete(id) {
  const res = await fetch(`/api/notes/${id}`, { method: 'DELETE' });
  if (!res.ok) return;
  setNotes((prev) => prev.filter((n) => n._id !== id));
}
```

---

## Section 3 — Explain In Your Own Words

### Question 3.1 — The CORS Error

You are building a React + Express app. The React dev server runs on `localhost:3000` and Express runs on `localhost:5000`. You try to fetch data from Express and get this error in the browser console:

> Access to fetch at 'http://localhost:5000/api/notes' from origin 'http://localhost:3000' has been blocked by CORS policy

In **3–5 sentences**, explain what this error means to a teammate who has never seen it before. Then explain how you would fix it.

The browser thinks 3000 and 5000 are two different websites because the port is different. So React is not allowed to read data from Express unless Express says it is ok. `fetch` fails even if the server is fine.

Fix: on Express use the `cors` package and `app.use(cors())` before the routes. Or only allow `http://localhost:3000`.

---

### Question 3.2 — Two Ways to Fetch

Look at these two pieces of code:

```javascript
// Version A
function getUser(id) {
  return fetch(`/api/users/${id}`).then((res) => res.json());
}

// Version B
async function getUser(id) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}
```

Are they doing the same thing? Explain the difference (if any) to someone who only knows Version A.

They do the same thing. Both fetch the user and return json. B is just A but written with async/await. `await fetch` is like putting the next line inside `.then`.

---

## Section 4 — Small Coding Tasks

### Question 4.1 — Write a Middleware

Write an Express middleware function that logs the HTTP method, URL, and the time it took to process the request. Example output:

```
POST /api/notes — 23ms
```

It should work for all routes. Show where you would place it in your app (write the `app.use(...)` line).

```javascript
function logTime(req, res, next) {
  const start = Date.now();
  res.on('finish', () => {
    const ms = Date.now() - start;
    console.log(`${req.method} ${req.originalUrl} — ${ms}ms`);
  });
  next();
}

app.use(logTime); // before all routes
```

---

### Question 4.2 — Write a Utility Function

Write a function called `groupByTag` that takes an array of bookmark objects and returns an object where the keys are tags and the values are arrays of bookmarks with that tag. Bookmarks without a tag should be grouped under `"untagged"`.

**Input:**

```javascript
[
  { title: 'React docs', url: 'https://react.dev', tag: 'frontend' },
  { title: 'MDN', url: 'https://developer.mozilla.org', tag: 'frontend' },
  { title: 'Express guide', url: 'https://expressjs.com', tag: 'backend' },
  { title: 'Random link', url: 'https://example.com' },
];
```

**Expected output:**

```javascript
{
  frontend: [
    { title: "React docs", url: "https://react.dev", tag: "frontend" },
    { title: "MDN", url: "https://developer.mozilla.org", tag: "frontend" }
  ],
  backend: [
    { title: "Express guide", url: "https://expressjs.com", tag: "backend" }
  ],
  untagged: [
    { title: "Random link", url: "https://example.com" }
  ]
}
```

```javascript
function groupByTag(bookmarks) {
  const result = {};
  for (const bookmark of bookmarks) {
    const tag = bookmark.tag || 'untagged';
    if (!result[tag]) {
      result[tag] = [];
    }
    result[tag].push(bookmark);
  }
  return result;
}
```
