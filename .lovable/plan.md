

# Phase 2: Labels, Due Dates, and Comments

Enhance cards with richer metadata -- color-coded labels, due dates with visual indicators, and a comment thread per card.

---

## 1. Database Changes (Migration)

### Labels
- Create a `labels` table: `id`, `board_id`, `name`, `color` (hex string), `created_at`
- Create a `card_labels` join table: `id`, `card_id`, `label_id` (many-to-many)
- RLS on both tables: board members only (using `is_board_member` helper)

### Due Dates
- Add a `due_date` column (timestamptz, nullable) to the existing `cards` table

### Comments
- Create a `comments` table: `id`, `card_id`, `user_id`, `content` (text), `created_at`, `updated_at`
- RLS: board members can view/create; users can only update/delete their own comments

### Realtime (optional)
- Enable realtime on `comments` table so collaborators see new comments instantly

---

## 2. Labels Feature

### Board-Level Label Management
- Add a "Manage Labels" option in the board header or a settings dropdown
- Dialog to create, edit, and delete labels (name + color picker from preset palette)
- Each board has its own set of labels

### Card Label Assignment
- In the card detail modal, add a "Labels" section
- Show available board labels as colored chips; click to toggle on/off
- Selected labels saved to `card_labels` join table

### Card Preview
- Show assigned label colors as small colored dots or pills on the card face in the column view

---

## 3. Due Dates Feature

### Card Detail Modal
- Add a "Due date" field using a date picker (react-day-picker, already installed)
- Allow clearing the due date
- Saved directly to the `due_date` column on `cards`

### Visual Indicators on Cards
- Show due date on the card face as a small badge
- Color-code: grey for future, yellow for due soon (within 24h), red for overdue

---

## 4. Comments Feature

### Comment Thread in Card Modal
- Below the description in the card detail dialog, add a "Comments" section
- Text input to post a new comment
- List of existing comments showing author name/avatar (from `profiles`), timestamp, and content
- Edit and delete buttons on the user's own comments

### Data Fetching
- Fetch comments for the active card with a dedicated query (joined with `profiles` for author info)
- Invalidate on create/update/delete

---

## 5. UI Updates to KanbanCard

The card face (in the column) will be updated to show:
1. Label color dots/pills at the top
2. Card title
3. Card description preview (existing)
4. Due date badge at the bottom (if set)
5. Comment count icon (if comments exist)

---

## Technical Details

### New Files
- `src/components/kanban/CardLabels.tsx` -- label chips display and toggle UI
- `src/components/kanban/LabelManager.tsx` -- board-level label CRUD dialog
- `src/components/kanban/CardDueDate.tsx` -- date picker component for due dates
- `src/components/kanban/CardComments.tsx` -- comment thread component

### Modified Files
- `src/components/kanban/KanbanCard.tsx` -- add labels, due date badge, comment count to card face; expand detail modal with labels, due date picker, and comments section
- `src/pages/BoardView.tsx` -- add "Manage Labels" button; pass board labels data down to columns/cards
- `src/components/kanban/KanbanColumn.tsx` -- pass labels data to cards

### Migration SQL (single migration)
```sql
-- Labels
CREATE TABLE public.labels (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  board_id uuid NOT NULL REFERENCES public.boards(id) ON DELETE CASCADE,
  name text NOT NULL DEFAULT 'Label',
  color text NOT NULL DEFAULT '#3b82f6',
  created_at timestamptz NOT NULL DEFAULT now()
);
ALTER TABLE public.labels ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Board members can manage labels" ON public.labels FOR ALL USING (is_board_member(auth.uid(), board_id)) WITH CHECK (is_board_member(auth.uid(), board_id));

-- Card-Label join
CREATE TABLE public.card_labels (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id uuid NOT NULL REFERENCES public.cards(id) ON DELETE CASCADE,
  label_id uuid NOT NULL REFERENCES public.labels(id) ON DELETE CASCADE,
  UNIQUE(card_id, label_id)
);
ALTER TABLE public.card_labels ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Board members can manage card labels" ON public.card_labels FOR ALL
  USING (EXISTS (SELECT 1 FROM cards c JOIN lists l ON c.list_id = l.id WHERE c.id = card_id AND is_board_member(auth.uid(), l.board_id)))
  WITH CHECK (EXISTS (SELECT 1 FROM cards c JOIN lists l ON c.list_id = l.id WHERE c.id = card_id AND is_board_member(auth.uid(), l.board_id)));

-- Due date on cards
ALTER TABLE public.cards ADD COLUMN due_date timestamptz;

-- Comments
CREATE TABLE public.comments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id uuid NOT NULL REFERENCES public.cards(id) ON DELETE CASCADE,
  user_id uuid NOT NULL,
  content text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
ALTER TABLE public.comments ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Board members can view comments" ON public.comments FOR SELECT
  USING (EXISTS (SELECT 1 FROM cards c JOIN lists l ON c.list_id = l.id WHERE c.id = card_id AND is_board_member(auth.uid(), l.board_id)));
CREATE POLICY "Board members can create comments" ON public.comments FOR INSERT
  WITH CHECK (auth.uid() = user_id AND EXISTS (SELECT 1 FROM cards c JOIN lists l ON c.list_id = l.id WHERE c.id = card_id AND is_board_member(auth.uid(), l.board_id)));
CREATE POLICY "Users can update own comments" ON public.comments FOR UPDATE
  USING (auth.uid() = user_id);
CREATE POLICY "Users can delete own comments" ON public.comments FOR DELETE
  USING (auth.uid() = user_id);

-- Updated_at trigger for comments
CREATE TRIGGER update_comments_updated_at BEFORE UPDATE ON public.comments
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### Queries Pattern
- Labels: fetched per board alongside lists
- Card labels: fetched in bulk for all cards on the board (single query with join)
- Comments: fetched per card when the detail modal opens
- Profiles joined to comments for author display name and avatar

