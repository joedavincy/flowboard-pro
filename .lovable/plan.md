

# Phase 1: Kanban Board Foundation

Build the core Kanban board with authentication, user profiles, and drag-and-drop board management using Supabase as the backend.

---

## 1. Supabase Setup & Authentication
- Connect to Supabase (Lovable Cloud)
- Email/password sign up and log in pages
- Auth context provider wrapping the app
- Protected routes — redirect unauthenticated users to login

## 2. User Profiles
- `profiles` table linked to `auth.users` with display name and avatar URL
- Database trigger to auto-create a profile on signup
- Profile edit page where users can update their name and avatar
- Avatar upload using Supabase Storage (public `avatars` bucket)
- RLS: users can only read/update their own profile

## 3. User Roles (Security Foundation)
- `app_role` enum (`admin`, `user`)
- `user_roles` table with RLS enabled
- `has_role()` security definer function to prevent RLS recursion
- Default role assigned on signup via trigger

## 4. Boards
- `boards` table with title, background color, and owner
- `board_members` table with role per board (`admin`, `member`)
- Dashboard page showing all boards the user belongs to
- Create, rename, and delete boards
- Board owner automatically added as admin member
- RLS: only board members can view/edit their boards

## 5. Lists (Columns)
- `lists` table with title, board reference, and `position` (integer for ordering)
- Create, rename, and delete lists within a board
- Drag-and-drop reordering of lists using @dnd-kit
- Position updates persisted to Supabase on drop
- RLS: accessible to board members only

## 6. Cards (Tasks)
- `cards` table with title, description, list reference, and `position` for ordering
- Create, edit, and delete cards within a list
- Card detail modal for editing title and description
- Drag-and-drop cards within a list and between lists using @dnd-kit
- Position and list updates persisted on drop
- RLS: accessible to board members only

## 7. Board View UI
- Full board view with horizontal scrollable columns
- Each column shows its cards in order
- "Add list" button at the end of columns
- "Add card" button at the bottom of each list
- Smooth drag-and-drop animations with @dnd-kit's sortable preset
- Toast notifications for create/delete actions

## 8. Navigation & Layout
- Top navigation bar with app logo, user avatar, and logout
- Dashboard as the home page (list of boards)
- Board detail page at `/board/:id`
- 404 page for invalid routes
- Responsive layout (desktop-first, works on tablet)

