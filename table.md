-- ============================================================
-- DevSphere — Initial schema migration
-- Run this in the Supabase SQL editor, or via `supabase db push`
-- ============================================================

-- ---------- Extensions ----------
create extension if not exists "pgcrypto"; -- for gen_random_uuid()

-- ---------- Enums ----------
create type user_role as enum ('student', 'advisor', 'admin');
create type verification_status as enum ('pending', 'verified', 'rejected');
create type verification_target as enum ('project', 'hackathon', 'certificate');
create type reminder_target as enum ('project', 'hackathon', 'certificate', 'poll');
create type announcement_target as enum ('all', 'students', 'advisors');
create type poll_status as enum ('open', 'closed');
create type form_field_type as enum ('text', 'number', 'select', 'file');
create type notification_type as enum ('hackathon_posted', 'verification_update', 'reminder', 'announcement');

-- ============================================================
-- Core identity
-- ============================================================

create table public.class_groups (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  department text,
  batch text,
  created_by uuid references auth.users(id),
  created_at timestamptz not null default now()
);

create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text not null,
  role user_role not null default 'student',
  class_group_id uuid references public.class_groups(id),
  department text,
  avatar_url text,
  github_url text,
  linkedin_url text,
  portfolio_url text,
  leetcode_username text,
  roll_no text,
  batch text,
  created_at timestamptz not null default now()
);

create table public.class_group_advisors (
  class_group_id uuid references public.class_groups(id) on delete cascade,
  advisor_id uuid references public.profiles(id) on delete cascade,
  subject text not null,
  primary key (class_group_id, advisor_id, subject)
);

-- Security-definer helper: avoids RLS recursion when policies need to check role
create or replace function public.current_role()
returns user_role
language sql
stable
security definer
set search_path = public
as $$
  select role from public.profiles where id = auth.uid();
$$;

-- Security-definer helper: returns the class_group_ids an advisor teaches
create or replace function public.advisor_class_groups()
returns setof uuid
language sql
stable
security definer
set search_path = public
as $$
  select class_group_id from public.class_group_advisors where advisor_id = auth.uid();
$$;

-- ============================================================
-- Domain tables
-- ============================================================

create table public.projects (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references public.profiles(id) on delete cascade,
  title text not null,
  description text,
  tech_stack text[],
  github_link text,
  demo_link text,
  current_status verification_status not null default 'pending',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.hackathons (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references public.profiles(id) on delete cascade,
  name text not null,
  event_date date,
  result text,
  certificate_url text,
  current_status verification_status not null default 'pending',
  created_at timestamptz not null default now()
);

create table public.certificates (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references public.profiles(id) on delete cascade,
  title text not null,
  issuer text,
  issue_date date,
  file_url text,
  current_status verification_status not null default 'pending',
  created_at timestamptz not null default now()
);

create table public.achievements (
  id uuid primary key default gen_random_uuid(),
  student_id uuid not null references public.profiles(id) on delete cascade,
  title text not null,
  description text,
  date_achieved date,
  created_at timestamptz not null default now()
);

create table public.verifications (
  id uuid primary key default gen_random_uuid(),
  target_type verification_target not null,
  target_id uuid not null,
  student_id uuid not null references public.profiles(id) on delete cascade,
  status verification_status not null,
  verified_by uuid references public.profiles(id),
  reason text,
  created_at timestamptz not null default now()
);
create index on public.verifications (target_type, target_id);

-- ============================================================
-- Communication
-- ============================================================

create table public.announcements (
  id uuid primary key default gen_random_uuid(),
  author_id uuid not null references public.profiles(id),
  title text not null,
  body text not null,
  target_role announcement_target not null default 'all',
  target_advisor_id uuid references public.profiles(id),
  attachment_url text,
  pinned boolean not null default false,
  created_at timestamptz not null default now()
);

create table public.polls (
  id uuid primary key default gen_random_uuid(),
  advisor_id uuid not null references public.profiles(id),
  question text not null,
  options jsonb not null,
  deadline timestamptz,
  status poll_status not null default 'open',
  created_at timestamptz not null default now()
);

create table public.poll_responses (
  id uuid primary key default gen_random_uuid(),
  poll_id uuid not null references public.polls(id) on delete cascade,
  student_id uuid not null references public.profiles(id) on delete cascade,
  selected_option text not null,
  responded_at timestamptz not null default now(),
  unique (poll_id, student_id)
);

create table public.reminders_log (
  id uuid primary key default gen_random_uuid(),
  target_type reminder_target not null,
  target_id uuid not null,
  student_id uuid not null references public.profiles(id) on delete cascade,
  reminder_count int not null default 0,
  last_reminded_at timestamptz,
  escalated boolean not null default false,
  escalated_at timestamptz
);
create index on public.reminders_log (target_type, target_id);

-- ============================================================
-- Hackathon postings + dynamic forms
-- ============================================================

create table public.hackathon_postings (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  description text,
  created_by uuid not null references public.profiles(id),
  registration_deadline timestamptz,
  published_at timestamptz not null default now()
);

create table public.hackathon_form_fields (
  id uuid primary key default gen_random_uuid(),
  hackathon_posting_id uuid not null references public.hackathon_postings(id) on delete cascade,
  field_label text not null,
  field_type form_field_type not null,
  field_options jsonb,
  required boolean not null default false,
  "order" int not null default 0
);

create table public.hackathon_form_responses (
  id uuid primary key default gen_random_uuid(),
  hackathon_posting_id uuid not null references public.hackathon_postings(id) on delete cascade,
  student_id uuid not null references public.profiles(id) on delete cascade,
  response_data jsonb not null,
  submitted_at timestamptz not null default now(),
  unique (hackathon_posting_id, student_id)
);

create table public.notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.profiles(id) on delete cascade,
  type notification_type not null,
  reference_id uuid,
  read_at timestamptz,
  created_at timestamptz not null default now()
);
create index on public.notifications (user_id, read_at);

-- ============================================================
-- RLS — enable on every table
-- ============================================================

alter table public.class_groups enable row level security;
alter table public.profiles enable row level security;
alter table public.class_group_advisors enable row level security;
alter table public.projects enable row level security;
alter table public.hackathons enable row level security;
alter table public.certificates enable row level security;
alter table public.achievements enable row level security;
alter table public.verifications enable row level security;
alter table public.announcements enable row level security;
alter table public.polls enable row level security;
alter table public.poll_responses enable row level security;
alter table public.reminders_log enable row level security;
alter table public.hackathon_postings enable row level security;
alter table public.hackathon_form_fields enable row level security;
alter table public.hackathon_form_responses enable row level security;
alter table public.notifications enable row level security;

-- ---------- profiles ----------
create policy "profiles_select" on public.profiles for select
using (
  id = auth.uid()
  or class_group_id in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);

create policy "profiles_update_self" on public.profiles for update
using (id = auth.uid());

-- Inserts/role changes/class_group_id changes happen via FastAPI using the
-- service role key (bypasses RLS entirely) — no student/advisor insert policy needed.

-- ---------- class_groups ----------
create policy "class_groups_select" on public.class_groups for select
using (
  id in (select public.advisor_class_groups())
  or id = (select class_group_id from public.profiles where id = auth.uid())
  or public.current_role() = 'admin'
);

-- ---------- class_group_advisors ----------
create policy "class_group_advisors_select" on public.class_group_advisors for select
using (
  advisor_id = auth.uid()
  or public.current_role() = 'admin'
  or class_group_id = (select class_group_id from public.profiles where id = auth.uid())
);

-- ---------- projects / hackathons / certificates (same shape) ----------
create policy "projects_select" on public.projects for select
using (
  student_id = auth.uid()
  or (select class_group_id from public.profiles where id = student_id) in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);
create policy "projects_insert" on public.projects for insert
with check (student_id = auth.uid());
create policy "projects_update_own" on public.projects for update
using (student_id = auth.uid());

create policy "hackathons_select" on public.hackathons for select
using (
  student_id = auth.uid()
  or (select class_group_id from public.profiles where id = student_id) in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);
create policy "hackathons_insert" on public.hackathons for insert
with check (student_id = auth.uid());

create policy "certificates_select" on public.certificates for select
using (
  student_id = auth.uid()
  or (select class_group_id from public.profiles where id = student_id) in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);
create policy "certificates_insert" on public.certificates for insert
with check (student_id = auth.uid());

-- ---------- achievements ----------
create policy "achievements_select" on public.achievements for select
using (
  student_id = auth.uid()
  or (select class_group_id from public.profiles where id = student_id) in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);
create policy "achievements_insert" on public.achievements for insert
with check (student_id = auth.uid());

-- ---------- verifications ----------
create policy "verifications_select" on public.verifications for select
using (
  student_id = auth.uid()
  or (select class_group_id from public.profiles where id = student_id) in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);
-- Inserts happen only via FastAPI (service role) through record_verification() — no client insert policy.

-- ---------- announcements ----------
create policy "announcements_select" on public.announcements for select
using (true); -- visible to all authenticated users; target_role filtering happens in FastAPI query logic

create policy "announcements_insert" on public.announcements for insert
with check (public.current_role() in ('advisor', 'admin'));

-- ---------- polls ----------
create policy "polls_select" on public.polls for select
using (
  advisor_id = auth.uid()
  or public.current_role() = 'admin'
  or (select class_group_id from public.profiles where id = auth.uid()) in (
    select class_group_id from public.class_group_advisors where advisor_id = polls.advisor_id
  )
);
create policy "polls_insert" on public.polls for insert
with check (public.current_role() = 'advisor');

-- ---------- poll_responses ----------
create policy "poll_responses_select" on public.poll_responses for select
using (
  student_id = auth.uid()
  or public.current_role() in ('advisor', 'admin')
);
create policy "poll_responses_insert" on public.poll_responses for insert
with check (student_id = auth.uid());

-- ---------- reminders_log ----------
create policy "reminders_log_select" on public.reminders_log for select
using (
  student_id = auth.uid()
  or (select class_group_id from public.profiles where id = student_id) in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);
-- Inserts only via FastAPI/background job (service role).

-- ---------- hackathon_postings ----------
create policy "hackathon_postings_select" on public.hackathon_postings for select
using (true); -- published postings are visible to everyone

create policy "hackathon_postings_insert" on public.hackathon_postings for insert
with check (public.current_role() = 'admin');

-- ---------- hackathon_form_fields ----------
create policy "hackathon_form_fields_select" on public.hackathon_form_fields for select
using (true);

-- ---------- hackathon_form_responses ----------
create policy "hackathon_form_responses_select" on public.hackathon_form_responses for select
using (
  student_id = auth.uid()
  or (select class_group_id from public.profiles where id = student_id) in (select public.advisor_class_groups())
  or public.current_role() = 'admin'
);
create policy "hackathon_form_responses_insert" on public.hackathon_form_responses for insert
with check (student_id = auth.uid());

-- ---------- notifications ----------
create policy "notifications_select" on public.notifications for select
using (user_id = auth.uid());
create policy "notifications_update_own" on public.notifications for update
using (user_id = auth.uid()); -- so a student can mark their own notification as read
-- Inserts only via FastAPI (service role) when fanning out notifications.