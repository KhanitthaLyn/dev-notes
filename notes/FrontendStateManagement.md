# Question

Where should search filters live?
• React State URL
• Local Storage
• Depends

# Answer

# The Big Picture & Analogy

Imagine walking into a library and telling the librarian: "I want mystery novels, published after 2020, under $10." The question is: **where should this search criteria be "remembered"?**

- If it's only remembered **in your head** (React State) — the moment you leave the library and come back, it's gone. You have to repeat it every time.
- If it's written on **the order slip you hand to the librarian** (URL) — you can photograph that slip and send it to a friend; they hand it to a different librarian and get the exact same result.
- If it's jotted in **your personal notebook** (Local Storage) — next time you visit this same library, the system remembers your usual preferences automatically, even after closing your laptop entirely.

All three storage locations are "correct" — they just solve different problems.

# Why Do We Need It? (Why This Distinction Matters)

The common mistake is developers defaulting to **React State for everything**, without thinking through the actual use case. The result:
- A user is filtering products, hits **refresh (F5), and all filters disappear** — starting over from scratch
- A user wants to **share a link to their filtered search** with a friend (e.g. "check out these shoes, size 8, under $50"), but can't — because the filter only lives in their own browser's memory
- A user closes the browser, reopens it, **expecting a remembered preference** (e.g. "always sort by price ascending"), but the system has forgotten everything

This is exactly why the answer is "Depends" — each storage location **solves a different problem.**

# Core Logic & How It Works

Let's look at each option and its true "job":

**🔹 React State (useState/useReducer)**
```jsx
const [filters, setFilters] = useState({ category: 'shoes', maxPrice: 2000 });
```
- Lives only in component memory — **gone instantly on refresh or tab close**
- Good for **UI-only, transient data that doesn't need to survive page transitions** — e.g. "is this modal open," "hover state"

**🔹 URL (Query Parameters)**
```
/products?category=shoes&maxPrice=2000&sort=price_asc
```
- Persists as long as the URL exists — **survives refresh, shareable via link, browser back/forward works correctly**
- Good for **state that's part of "what this page actually is"** — it belongs in the URL because it defines the page's identity (bookmarkable state)

**🔹 Local Storage**
```js
localStorage.setItem('preferredSort', 'price_asc');
```
- Persists across sessions, browser tabs, and days — until explicitly cleared
- Good for **user preferences unrelated to "what this page is"** but tied to "what this particular user likes" — e.g. selected language, theme (dark/light mode), display settings that should stay the same every time they return

# Trade-offs & When to Use (Decision Framework)

Here's a set of questions to ask yourself before choosing:

| Question | If "yes" → choose |
|---|---|
| Should sharing this link show someone else the exact same result? | **URL** |
| Should the browser's "Back" button restore the previous filter state? | **URL** |
| Is this filter part of "what this page is" (bookmarkable)? | **URL** |
| Does the user want the system to remember this preference next time (across sessions)? | **Local Storage** |
| Is it a user-specific setting unrelated to the current page (like theme)? | **Local Storage** |
| Is it just transient UI state that doesn't need to survive a refresh? | **React State** |
| Is the data sensitive and shouldn't be visible/shareable in a URL? | **React State or server-side session** |

**Key trade-offs:**
- **URL:** pro is shareable/bookmarkable, but con is **everything is fully visible** (not suitable for sensitive data), and URLs get unwieldy if filters get too complex
- **Local Storage:** pro is persistence across sessions, but con is **no sync across devices** (opening from a different browser or phone won't see the same preference), and it shouldn't hold sensitive data since it's unencrypted
- **React State:** fastest and simplest, but **disappears instantly on reload**, which can be a major UX problem for ecommerce, where users expect filters to persist

**One more important point: often you need "both together," not just one choice**
```jsx
// URL is the primary source of truth
// but sync into React State so the UI can re-render quickly without re-parsing the URL every time
const [searchParams, setSearchParams] = useSearchParams(); // URL-driven
const filters = useMemo(() => parseFilters(searchParams), [searchParams]);

// Local Storage only supplies a "default preference" when the user hasn't chosen anything yet
useEffect(() => {
  if (!searchParams.has('sort')) {
    const savedSort = localStorage.getItem('preferredSort');
    if (savedSort) setSearchParams({ ...filters, sort: savedSort });
  }
}, []);
```

# Real-World Scenario (Ecommerce Domain)

A typical ecommerce "product search" page:

```
/products?category=shoes&size=40&maxPrice=2000&sort=price_asc&page=2
```

**Why it needs to be in the URL:**
- A user copies this link into a family group chat saying "shoes I want to buy" — the friend clicks it and sees **the exact same results**, no need to re-filter
- A user clicks "Back" from a Product Detail page to return to the List page — **filters need to still be there**, not reset to a fresh page
- Google/SEO can index different filtered pages separately (e.g. "size 40 shoes" is a distinct page search engines can see, separate from "size 41 shoes")

**Why "dark/light mode theme" belongs in Local Storage instead:**
- It's not related to "what this page is" — it's a personal user preference that should follow them across every page on the site, not just the product list page
- It shouldn't be shareable — if a URL with `?theme=dark` gets sent to a friend, they shouldn't be forced into dark mode just because you were using it

# Lead's Key Takeaway

> **"Always ask yourself: 'is this state part of this page's identity (URL), a personal user preference (Storage), or just transient UI (React State)?' — that question answers where it should live."**
>
> A good Lead doesn't treat state management as just "which library to use" (Redux, Zustand, Context). They always ask first: **"how long should this state outlive the component, and how shareable/persistent does it need to be?"** — because getting the storage choice wrong from the start becomes one of the most complained-about UX bugs (lost filters, unshareable links, forgotten preferences).
