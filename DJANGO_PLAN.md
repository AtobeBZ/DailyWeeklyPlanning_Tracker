# Django Implementation Plan - Daily Clock Planner

## Overview

Convert the current single-file HTML application into a full-stack Django web application with:
- Django templates for server-rendered pages
- HTMX for dynamic interactions without page reloads
- Alpine.js for small client-side interactivity
- PostgreSQL for data persistence
- Django built-in auth + django-allauth for SSO

---

## 1. Project Structure

```
dailyplanner/
├── manage.py
├── requirements.txt
├── .env                          # Environment variables (SECRET_KEY, DB, etc.)
├── .gitignore
│
├── config/                       # Project configuration
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py              # Shared settings
│   │   ├── development.py       # Dev settings (DEBUG=True)
│   │   └── production.py        # Prod settings (DEBUG=False)
│   ├── urls.py                  # Root URL configuration
│   ├── wsgi.py
│   └── asgi.py                  # For Django Channels (future)
│
├── apps/
│   ├── __init__.py
│   │
│   ├── accounts/                # User management
│   │   ├── __init__.py
│   │   ├── models.py            # Custom User model (optional)
│   │   ├── views.py             # Login, register, profile
│   │   ├── urls.py
│   │   ├── forms.py
│   │   └── templates/
│   │       └── accounts/
│   │           ├── login.html
│   │           ├── register.html
│   │           └── profile.html
│   │
│   ├── planner/                 # Core planning functionality
│   │   ├── __init__.py
│   │   ├── models.py            # Templates, Activities, DayOverrides
│   │   ├── views.py             # Daily, Weekly, Year views
│   │   ├── urls.py
│   │   ├── forms.py             # Activity forms
│   │   ├── api.py               # HTMX endpoints (partial HTML responses)
│   │   ├── services.py          # Business logic (activity resolution)
│   │   ├── templatetags/
│   │   │   ├── __init__.py
│   │   │   └── planner_tags.py  # Custom template filters
│   │   └── templates/
│   │       └── planner/
│   │           ├── base.html           # Base template with nav
│   │           ├── daily.html          # Daily clock view
│   │           ├── weekly.html         # Weekly Outlook-style view
│   │           ├── year.html           # Year calendar view
│   │           ├── onboarding.html     # Onboarding wizard
│   │           └── partials/           # HTMX partial templates
│   │               ├── _clock.html
│   │               ├── _activity_list.html
│   │               ├── _activity_modal.html
│   │               ├── _weekly_grid.html
│   │               ├── _weekly_day_column.html
│   │               ├── _month_card.html
│   │               └── _stats.html
│   │
│   └── core/                    # Shared utilities
│       ├── __init__.py
│       ├── mixins.py            # View mixins (LoginRequired, etc.)
│       └── utils.py             # Helper functions
│
├── static/
│   ├── css/
│   │   ├── main.css             # Global styles (from current app)
│   │   ├── clock.css            # Clock-specific styles
│   │   ├── weekly.css           # Weekly view styles
│   │   └── year.css             # Year view styles
│   ├── js/
│   │   ├── htmx.min.js          # HTMX library
│   │   ├── alpine.min.js        # Alpine.js
│   │   ├── chart.min.js         # Chart.js for visualizations
│   │   ├── clock.js             # Clock rendering logic
│   │   └── app.js               # Global JS utilities
│   └── images/
│       └── logo.svg
│
├── templates/                   # Global templates
│   ├── base.html                # Root base template
│   ├── components/
│   │   ├── _navbar.html
│   │   ├── _footer.html
│   │   ├── _modal.html
│   │   └── _toast.html
│   └── errors/
│       ├── 404.html
│       └── 500.html
│
└── tests/
    ├── __init__.py
    ├── test_models.py
    ├── test_views.py
    └── test_services.py
```

---

## 2. Data Models

### 2.1 Core Models (apps/planner/models.py)

```python
from django.db import models
from django.contrib.auth import get_user_model
from django.core.validators import MinValueValidator, MaxValueValidator

User = get_user_model()


class Category(models.Model):
    """User-customizable activity categories"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='categories')
    name = models.CharField(max_length=50)  # 'work', 'family', 'admin', etc.
    label = models.CharField(max_length=50)  # 'Work', 'Family', 'Admin', etc.
    color = models.CharField(max_length=7)  # Hex color '#e74c3c'
    sort_order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['sort_order', 'name']
        unique_together = ['user', 'name']

    def __str__(self):
        return f"{self.user.username} - {self.label}"


class DayType(models.Model):
    """Day type templates (Work Day, Off Day, etc.)"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='day_types')
    name = models.CharField(max_length=50)  # 'Work Day', 'Off Day', 'Holiday'
    is_work_like = models.BooleanField(default=True)  # True = uses weekday templates
    is_default = models.BooleanField(default=False)  # Default day type for new days
    sort_order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['sort_order', 'name']
        unique_together = ['user', 'name']

    def __str__(self):
        return f"{self.user.username} - {self.name}"


class DayTemplate(models.Model):
    """Base template for a day type (e.g., 'Work Day' template)"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='day_templates')
    day_type = models.OneToOneField(DayType, on_delete=models.CASCADE, related_name='template')

    class Meta:
        unique_together = ['user', 'day_type']

    def __str__(self):
        return f"{self.user.username} - {self.day_type.name} Template"


class Activity(models.Model):
    """An activity block within a template"""
    template = models.ForeignKey(DayTemplate, on_delete=models.CASCADE, related_name='activities')
    name = models.CharField(max_length=100)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    start_time = models.TimeField()
    end_time = models.TimeField()
    color = models.CharField(max_length=7, blank=True)  # Custom color override
    sort_order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['start_time', 'sort_order']

    def __str__(self):
        return f"{self.template} - {self.name} ({self.start_time}-{self.end_time})"

    @property
    def duration_minutes(self):
        """Calculate duration in minutes (handles overnight)"""
        start = self.start_time.hour * 60 + self.start_time.minute
        end = self.end_time.hour * 60 + self.end_time.minute
        if end < start:  # Overnight activity
            return (1440 - start) + end
        return end - start

    @property
    def is_overnight(self):
        """Check if activity spans midnight"""
        start = self.start_time.hour * 60 + self.start_time.minute
        end = self.end_time.hour * 60 + self.end_time.minute
        return end < start


class WeekdayConfig(models.Model):
    """Configuration for each weekday (which day type + custom activities)"""
    WEEKDAY_CHOICES = [
        (0, 'Monday'),
        (1, 'Tuesday'),
        (2, 'Wednesday'),
        (3, 'Thursday'),
        (4, 'Friday'),
        (5, 'Saturday'),
        (6, 'Sunday'),
    ]

    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='weekday_configs')
    weekday = models.IntegerField(choices=WEEKDAY_CHOICES)
    base_day_type = models.ForeignKey(DayType, on_delete=models.CASCADE)
    has_custom = models.BooleanField(default=False)  # True if has custom activities

    class Meta:
        unique_together = ['user', 'weekday']
        ordering = ['weekday']

    def __str__(self):
        return f"{self.user.username} - {self.get_weekday_display()} ({self.base_day_type.name})"


class WeekdayActivity(models.Model):
    """Custom activity for a specific weekday (overrides template)"""
    weekday_config = models.ForeignKey(WeekdayConfig, on_delete=models.CASCADE, related_name='custom_activities')
    name = models.CharField(max_length=100)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    start_time = models.TimeField()
    end_time = models.TimeField()
    color = models.CharField(max_length=7, blank=True)
    sort_order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['start_time', 'sort_order']

    def __str__(self):
        return f"{self.weekday_config} - {self.name}"


class DayOverride(models.Model):
    """Override day type for a specific date (holidays, vacation, etc.)"""
    PERIOD_CHOICES = [
        ('full', 'Full Day'),
        ('am', 'AM Only'),
        ('pm', 'PM Only'),
    ]

    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='day_overrides')
    date = models.DateField()
    day_type = models.ForeignKey(DayType, on_delete=models.CASCADE)
    period = models.CharField(max_length=4, choices=PERIOD_CHOICES, default='full')
    note = models.CharField(max_length=200, blank=True)  # e.g., "Christmas"

    class Meta:
        unique_together = ['user', 'date', 'period']
        ordering = ['date']

    def __str__(self):
        return f"{self.user.username} - {self.date} ({self.day_type.name})"


class PublicHoliday(models.Model):
    """Pre-loaded public holidays by region"""
    region = models.CharField(max_length=50)  # 'geneva', 'zurich', 'france', etc.
    date = models.DateField()
    name = models.CharField(max_length=100)

    class Meta:
        unique_together = ['region', 'date']
        ordering = ['date']

    def __str__(self):
        return f"{self.region} - {self.date} - {self.name}"


class UserSettings(models.Model):
    """User preferences and settings"""
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='settings')
    region = models.CharField(max_length=50, default='geneva')  # For public holidays
    theme = models.CharField(max_length=20, default='light')  # 'light', 'dark'
    week_start = models.IntegerField(default=0)  # 0=Monday, 6=Sunday
    default_view = models.CharField(max_length=20, default='weekly')  # 'daily', 'weekly', 'year'
    onboarding_completed = models.BooleanField(default=False)

    def __str__(self):
        return f"Settings for {self.user.username}"
```

### 2.2 Model Relationships Diagram

```
User (Django Auth)
 │
 ├── UserSettings (1:1)
 │
 ├── Category (1:N)
 │    └── color, label
 │
 ├── DayType (1:N)
 │    ├── name: "Work Day", "Off Day", etc.
 │    ├── is_work_like: true/false
 │    └── DayTemplate (1:1)
 │         └── Activity (1:N)
 │              └── name, start_time, end_time, category
 │
 ├── WeekdayConfig (1:7, one per weekday)
 │    ├── weekday: 0-6 (Mon-Sun)
 │    ├── base_day_type → DayType
 │    ├── has_custom: true/false
 │    └── WeekdayActivity (1:N, only if has_custom)
 │         └── name, start_time, end_time, category
 │
 └── DayOverride (1:N)
      ├── date: specific date
      ├── day_type → DayType
      └── period: 'full', 'am', 'pm'
```

---

## 3. URL Routes

### 3.1 Main URLs (config/urls.py)

```python
from django.contrib import admin
from django.urls import path, include
from django.contrib.auth import views as auth_views

urlpatterns = [
    path('admin/', admin.site.urls),

    # Authentication
    path('accounts/', include('apps.accounts.urls')),
    path('accounts/', include('allauth.urls')),  # Google/Apple SSO

    # Main app
    path('', include('apps.planner.urls')),
]
```

### 3.2 Planner URLs (apps/planner/urls.py)

```python
from django.urls import path
from . import views, api

app_name = 'planner'

urlpatterns = [
    # Main views (full page loads)
    path('', views.DashboardView.as_view(), name='dashboard'),
    path('daily/', views.DailyView.as_view(), name='daily'),
    path('weekly/', views.WeeklyView.as_view(), name='weekly'),
    path('year/', views.YearView.as_view(), name='year'),
    path('year/<int:year>/', views.YearView.as_view(), name='year_specific'),
    path('onboarding/', views.OnboardingView.as_view(), name='onboarding'),
    path('settings/', views.SettingsView.as_view(), name='settings'),

    # HTMX API endpoints (return partial HTML)
    path('api/daily/clock/', api.daily_clock, name='api_daily_clock'),
    path('api/daily/activities/', api.daily_activities, name='api_daily_activities'),
    path('api/daily/stats/', api.daily_stats, name='api_daily_stats'),

    path('api/weekly/grid/', api.weekly_grid, name='api_weekly_grid'),
    path('api/weekly/day/<str:weekday>/', api.weekly_day_column, name='api_weekly_day'),
    path('api/weekly/stats/', api.weekly_stats, name='api_weekly_stats'),

    path('api/year/calendar/', api.year_calendar, name='api_year_calendar'),
    path('api/year/month/<int:month>/', api.year_month, name='api_year_month'),

    # Activity CRUD (HTMX)
    path('api/activity/modal/', api.activity_modal, name='api_activity_modal'),
    path('api/activity/create/', api.activity_create, name='api_activity_create'),
    path('api/activity/<int:pk>/edit/', api.activity_edit, name='api_activity_edit'),
    path('api/activity/<int:pk>/update/', api.activity_update, name='api_activity_update'),
    path('api/activity/<int:pk>/delete/', api.activity_delete, name='api_activity_delete'),

    # Day type management
    path('api/daytype/<int:pk>/set/', api.set_day_type, name='api_set_day_type'),
    path('api/weekday/<str:weekday>/type/', api.set_weekday_type, name='api_set_weekday_type'),
    path('api/date/<str:date>/override/', api.set_date_override, name='api_set_date_override'),

    # Categories
    path('api/categories/', api.categories_list, name='api_categories'),
    path('api/category/<int:pk>/color/', api.update_category_color, name='api_category_color'),

    # Export/Import
    path('export/json/', views.export_json, name='export_json'),
    path('export/pdf/', views.export_pdf, name='export_pdf'),
    path('import/json/', views.import_json, name='import_json'),
]
```

---

## 4. Views

### 4.1 Main Views (apps/planner/views.py)

```python
from django.views.generic import TemplateView, View
from django.contrib.auth.mixins import LoginRequiredMixin
from django.shortcuts import redirect
from django.http import JsonResponse, HttpResponse
from .models import DayType, DayTemplate, WeekdayConfig, Category, UserSettings
from .services import ActivityResolver
import json


class DashboardView(LoginRequiredMixin, View):
    """Redirect to user's default view"""
    def get(self, request):
        settings = getattr(request.user, 'settings', None)
        if settings and not settings.onboarding_completed:
            return redirect('planner:onboarding')

        default_view = settings.default_view if settings else 'weekly'
        return redirect(f'planner:{default_view}')


class DailyView(LoginRequiredMixin, TemplateView):
    template_name = 'planner/daily.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['day_types'] = DayType.objects.filter(user=self.request.user)
        context['categories'] = Category.objects.filter(user=self.request.user)
        context['current_day_type'] = self.request.GET.get('type', 'Work Day')

        # Get activities for current day type
        resolver = ActivityResolver(self.request.user)
        context['activities'] = resolver.get_template_activities(context['current_day_type'])

        return context


class WeeklyView(LoginRequiredMixin, TemplateView):
    template_name = 'planner/weekly.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.filter(user=self.request.user)
        context['day_types'] = DayType.objects.filter(user=self.request.user)

        # Get weekday configs with activities
        resolver = ActivityResolver(self.request.user)
        context['weekdays'] = resolver.get_weekly_data()

        return context


class YearView(LoginRequiredMixin, TemplateView):
    template_name = 'planner/year.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        year = kwargs.get('year', timezone.now().year)
        context['year'] = year
        context['day_types'] = DayType.objects.filter(user=self.request.user)

        # Get year calendar data
        resolver = ActivityResolver(self.request.user)
        context['months'] = resolver.get_year_data(year)
        context['stats'] = resolver.get_year_stats(year)

        return context


class OnboardingView(LoginRequiredMixin, TemplateView):
    template_name = 'planner/onboarding.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['regions'] = [
            ('geneva', 'Geneva, Switzerland'),
            ('zurich', 'Zurich, Switzerland'),
            ('france', 'France'),
            ('germany', 'Germany'),
            ('uk', 'United Kingdom'),
            ('us', 'United States'),
        ]
        return context

    def post(self, request):
        # Process onboarding data and create initial setup
        # ... (create default categories, day types, templates)
        settings, _ = UserSettings.objects.get_or_create(user=request.user)
        settings.onboarding_completed = True
        settings.save()
        return redirect('planner:weekly')


class SettingsView(LoginRequiredMixin, TemplateView):
    template_name = 'planner/settings.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['settings'], _ = UserSettings.objects.get_or_create(user=self.request.user)
        context['categories'] = Category.objects.filter(user=self.request.user)
        context['day_types'] = DayType.objects.filter(user=self.request.user)
        return context


def export_json(request):
    """Export all user data as JSON"""
    resolver = ActivityResolver(request.user)
    data = resolver.export_all_data()

    response = HttpResponse(json.dumps(data, indent=2), content_type='application/json')
    response['Content-Disposition'] = 'attachment; filename="planner_export.json"'
    return response


def import_json(request):
    """Import user data from JSON"""
    if request.method == 'POST' and request.FILES.get('file'):
        data = json.load(request.FILES['file'])
        resolver = ActivityResolver(request.user)
        resolver.import_all_data(data)
        return redirect('planner:settings')
    return redirect('planner:settings')


def export_pdf(request):
    """Export weekly summary as PDF"""
    # Use weasyprint or reportlab
    pass
```

### 4.2 HTMX API Views (apps/planner/api.py)

```python
from django.http import HttpResponse
from django.shortcuts import render, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods
from .models import Activity, WeekdayActivity, WeekdayConfig, DayType, Category
from .forms import ActivityForm
from .services import ActivityResolver


@login_required
def weekly_grid(request):
    """Return the full weekly grid (HTMX partial)"""
    resolver = ActivityResolver(request.user)
    weekdays = resolver.get_weekly_data()
    categories = Category.objects.filter(user=request.user)
    day_types = DayType.objects.filter(user=request.user)

    return render(request, 'planner/partials/_weekly_grid.html', {
        'weekdays': weekdays,
        'categories': categories,
        'day_types': day_types,
    })


@login_required
def weekly_day_column(request, weekday):
    """Return a single day column (HTMX partial for updates)"""
    resolver = ActivityResolver(request.user)
    day_data = resolver.get_weekday_data(weekday)

    return render(request, 'planner/partials/_weekly_day_column.html', {
        'day': day_data,
    })


@login_required
def activity_modal(request):
    """Return activity edit/create modal content"""
    activity_id = request.GET.get('id')
    weekday = request.GET.get('weekday')
    day_type = request.GET.get('day_type')
    start_time = request.GET.get('start_time', '09:00')
    scope = request.GET.get('scope', 'weekday')

    if activity_id:
        # Edit existing
        activity = get_object_or_404(Activity, pk=activity_id)
        form = ActivityForm(instance=activity)
        is_new = False
    else:
        # Create new
        form = ActivityForm(initial={
            'start_time': start_time,
            'end_time': calculate_end_time(start_time, 60),
            'category': Category.objects.filter(user=request.user, name='work').first(),
        })
        is_new = True

    categories = Category.objects.filter(user=request.user)

    return render(request, 'planner/partials/_activity_modal.html', {
        'form': form,
        'activity_id': activity_id,
        'weekday': weekday,
        'day_type': day_type,
        'scope': scope,
        'is_new': is_new,
        'categories': categories,
    })


@login_required
@require_http_methods(["POST"])
def activity_create(request):
    """Create a new activity"""
    form = ActivityForm(request.POST)
    scope = request.POST.get('scope', 'weekday')
    weekday = request.POST.get('weekday')
    day_type_name = request.POST.get('day_type')

    if form.is_valid():
        activity = form.save(commit=False)

        if scope == 'weekday':
            # Add to weekday custom activities
            weekday_config = WeekdayConfig.objects.get(
                user=request.user,
                weekday=weekday_name_to_int(weekday)
            )
            if not weekday_config.has_custom:
                # Copy template activities to weekday first
                copy_template_to_weekday(weekday_config)

            weekday_activity = WeekdayActivity.objects.create(
                weekday_config=weekday_config,
                name=activity.name,
                category=activity.category,
                start_time=activity.start_time,
                end_time=activity.end_time,
                color=activity.color,
            )
        else:
            # Add to base template
            day_type = DayType.objects.get(user=request.user, name=day_type_name)
            template, _ = DayTemplate.objects.get_or_create(
                user=request.user,
                day_type=day_type
            )
            activity.template = template
            activity.save()

        # Return updated grid
        return weekly_grid(request)

    # Return form with errors
    return render(request, 'planner/partials/_activity_modal.html', {
        'form': form,
        'errors': form.errors,
    })


@login_required
@require_http_methods(["POST"])
def activity_update(request, pk):
    """Update an existing activity"""
    scope = request.POST.get('scope', 'weekday')

    if scope == 'weekday':
        activity = get_object_or_404(WeekdayActivity, pk=pk)
    else:
        activity = get_object_or_404(Activity, pk=pk)

    form = ActivityForm(request.POST, instance=activity)

    if form.is_valid():
        form.save()
        return weekly_grid(request)

    return render(request, 'planner/partials/_activity_modal.html', {
        'form': form,
        'errors': form.errors,
    })


@login_required
@require_http_methods(["DELETE", "POST"])
def activity_delete(request, pk):
    """Delete an activity"""
    scope = request.POST.get('scope', 'weekday')

    if scope == 'weekday':
        activity = get_object_or_404(WeekdayActivity, pk=pk)
    else:
        activity = get_object_or_404(Activity, pk=pk)

    activity.delete()
    return weekly_grid(request)


@login_required
@require_http_methods(["POST"])
def set_weekday_type(request, weekday):
    """Change the day type for a weekday"""
    day_type_name = request.POST.get('day_type')
    day_type = get_object_or_404(DayType, user=request.user, name=day_type_name)

    weekday_int = weekday_name_to_int(weekday)
    config, _ = WeekdayConfig.objects.get_or_create(
        user=request.user,
        weekday=weekday_int,
        defaults={'base_day_type': day_type}
    )
    config.base_day_type = day_type
    config.has_custom = False  # Reset custom when changing type
    config.save()

    # Clear custom activities
    config.custom_activities.all().delete()

    return weekly_day_column(request, weekday)


@login_required
@require_http_methods(["POST"])
def set_date_override(request, date):
    """Set or clear a day override for a specific date"""
    day_type_name = request.POST.get('day_type')
    period = request.POST.get('period', 'full')

    if day_type_name == 'clear':
        DayOverride.objects.filter(user=request.user, date=date).delete()
    else:
        day_type = get_object_or_404(DayType, user=request.user, name=day_type_name)
        DayOverride.objects.update_or_create(
            user=request.user,
            date=date,
            period=period,
            defaults={'day_type': day_type}
        )

    return year_calendar(request)


# Helper functions
def weekday_name_to_int(name):
    """Convert weekday name to integer (0=Monday)"""
    days = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
    return days.index(name)


def calculate_end_time(start_time, duration_minutes):
    """Calculate end time from start time and duration"""
    from datetime import datetime, timedelta
    start = datetime.strptime(start_time, '%H:%M')
    end = start + timedelta(minutes=duration_minutes)
    return end.strftime('%H:%M')


def copy_template_to_weekday(weekday_config):
    """Copy activities from base template to weekday custom"""
    template = weekday_config.base_day_type.template
    for activity in template.activities.all():
        WeekdayActivity.objects.create(
            weekday_config=weekday_config,
            name=activity.name,
            category=activity.category,
            start_time=activity.start_time,
            end_time=activity.end_time,
            color=activity.color,
        )
    weekday_config.has_custom = True
    weekday_config.save()
```

---

## 5. Service Layer (Business Logic)

### apps/planner/services.py

```python
from datetime import date, timedelta
from django.db.models import Sum
from .models import (
    DayType, DayTemplate, Activity,
    WeekdayConfig, WeekdayActivity,
    DayOverride, PublicHoliday, Category
)


class ActivityResolver:
    """
    Resolves which activities to show for any given day.
    Implements the 3-tier hierarchy:
    1. Day Type Templates (base)
    2. Weekday Templates (customization)
    3. Day Overrides (specific dates)
    """

    def __init__(self, user):
        self.user = user

    def get_activities_for_date(self, target_date):
        """
        Get the resolved activities for a specific date.
        Returns list of activity dicts.
        """
        # Step 1: Get effective day type (check overrides first)
        effective_type = self.get_effective_day_type(target_date)

        # Step 2: Determine if work-like or off-like
        if effective_type.is_work_like:
            # Check for weekday customization
            weekday = target_date.weekday()
            weekday_config = WeekdayConfig.objects.filter(
                user=self.user,
                weekday=weekday
            ).first()

            if weekday_config and weekday_config.has_custom:
                return list(weekday_config.custom_activities.all())

            # Fall back to base template
            return self.get_template_activities(effective_type.name)
        else:
            # Off-like days always use Off Day template (no weekday customs)
            return self.get_template_activities('Off Day')

    def get_effective_day_type(self, target_date):
        """Get the day type for a date, considering overrides"""
        # Check for explicit override
        override = DayOverride.objects.filter(
            user=self.user,
            date=target_date
        ).first()

        if override:
            return override.day_type

        # Check for public holiday
        settings = getattr(self.user, 'settings', None)
        region = settings.region if settings else 'geneva'

        holiday = PublicHoliday.objects.filter(
            region=region,
            date=target_date
        ).first()

        if holiday:
            return DayType.objects.get(user=self.user, name='Public Holiday')

        # Fall back to weekday default
        weekday = target_date.weekday()
        config = WeekdayConfig.objects.filter(
            user=self.user,
            weekday=weekday
        ).first()

        if config:
            return config.base_day_type

        # Default to Work Day for weekdays, Off Day for weekends
        if weekday < 5:
            return DayType.objects.get(user=self.user, name='Work Day')
        return DayType.objects.get(user=self.user, name='Off Day')

    def get_template_activities(self, day_type_name):
        """Get activities for a day type template"""
        try:
            day_type = DayType.objects.get(user=self.user, name=day_type_name)
            template = day_type.template
            return list(template.activities.all())
        except (DayType.DoesNotExist, DayTemplate.DoesNotExist):
            return []

    def get_weekly_data(self):
        """Get full week data for weekly view"""
        weekdays = []
        day_names = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']

        for i, name in enumerate(day_names):
            config = WeekdayConfig.objects.filter(user=self.user, weekday=i).first()

            if config:
                day_type = config.base_day_type
                has_custom = config.has_custom

                if has_custom:
                    activities = list(config.custom_activities.all())
                else:
                    activities = self.get_template_activities(day_type.name)
            else:
                # Default: Work Day for Mon-Fri, Off Day for Sat-Sun
                day_type_name = 'Work Day' if i < 5 else 'Off Day'
                day_type = DayType.objects.filter(user=self.user, name=day_type_name).first()
                has_custom = False
                activities = self.get_template_activities(day_type_name)

            weekdays.append({
                'index': i,
                'name': name,
                'day_type': day_type,
                'has_custom': has_custom,
                'activities': activities,
                'is_weekend': i >= 5,
            })

        return weekdays

    def get_year_data(self, year):
        """Get year calendar data"""
        months = []

        for month in range(1, 13):
            month_data = self.get_month_data(year, month)
            months.append(month_data)

        return months

    def get_month_data(self, year, month):
        """Get month calendar data with day types"""
        from calendar import monthrange, monthcalendar

        _, num_days = monthrange(year, month)
        cal = monthcalendar(year, month)

        days = []
        work_count = 0
        off_count = 0

        for day in range(1, num_days + 1):
            d = date(year, month, day)
            day_type = self.get_effective_day_type(d)

            if day_type.is_work_like:
                work_count += 1
            else:
                off_count += 1

            days.append({
                'day': day,
                'date': d,
                'day_type': day_type,
                'is_today': d == date.today(),
                'is_weekend': d.weekday() >= 5,
            })

        work_percent = round(work_count / num_days * 100)

        return {
            'month': month,
            'name': date(year, month, 1).strftime('%B'),
            'calendar': cal,
            'days': days,
            'work_count': work_count,
            'off_count': off_count,
            'work_percent': work_percent,
            'total_days': num_days,
        }

    def get_year_stats(self, year):
        """Calculate yearly statistics"""
        total_work = 0
        total_off = 0
        total_holidays = 0
        total_vacation = 0
        total_sick = 0

        start = date(year, 1, 1)
        end = date(year, 12, 31)
        current = start

        while current <= end:
            day_type = self.get_effective_day_type(current)

            if day_type.name == 'Work Day':
                total_work += 1
            elif day_type.name == 'Public Holiday':
                total_holidays += 1
            elif day_type.name == 'Vacation':
                total_vacation += 1
            elif day_type.name == 'Sick Day':
                total_sick += 1
            else:
                total_off += 1

            current += timedelta(days=1)

        return {
            'work_days': total_work,
            'off_days': total_off,
            'holidays': total_holidays,
            'vacation': total_vacation,
            'sick': total_sick,
            'total': (end - start).days + 1,
        }

    def export_all_data(self):
        """Export all user data as JSON-serializable dict"""
        # ... implementation
        pass

    def import_all_data(self, data):
        """Import data from JSON dict"""
        # ... implementation
        pass
```

---

## 6. Templates

### 6.1 Base Template (templates/base.html)

```html
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Daily Clock Planner{% endblock %}</title>

    <!-- Styles -->
    <link rel="stylesheet" href="{% static 'css/main.css' %}">
    {% block extra_css %}{% endblock %}

    <!-- HTMX -->
    <script src="{% static 'js/htmx.min.js' %}"></script>

    <!-- Alpine.js -->
    <script defer src="{% static 'js/alpine.min.js' %}"></script>

    <!-- HTMX Config -->
    <meta name="htmx-config" content='{"defaultSwapStyle":"innerHTML"}'>
</head>
<body hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>

    {% include 'components/_navbar.html' %}

    <main class="container">
        {% block content %}{% endblock %}
    </main>

    <!-- Toast notifications -->
    <div id="toast-container"></div>

    <!-- Modal container -->
    <div id="modal-container"
         x-data="{ open: false }"
         x-show="open"
         @modal-open.window="open = true"
         @modal-close.window="open = false"
         class="modal-overlay">
        <div id="modal-content" class="modal-content" @click.away="open = false">
            <!-- HTMX loads modal content here -->
        </div>
    </div>

    <!-- Tooltip -->
    <div id="tooltip" style="display: none;"></div>

    {% block extra_js %}{% endblock %}
</body>
</html>
```

### 6.2 Weekly View Template (apps/planner/templates/planner/weekly.html)

```html
{% extends 'base.html' %}
{% load static planner_tags %}

{% block title %}Weekly View - Daily Clock Planner{% endblock %}

{% block extra_css %}
<link rel="stylesheet" href="{% static 'css/weekly.css' %}">
{% endblock %}

{% block content %}
<div class="weekly-container" x-data="weeklyView()">

    <!-- Header -->
    <div class="weekly-header">
        <h1>Weekly Schedule</h1>
        <div class="view-toggle">
            <a href="{% url 'planner:daily' %}" class="btn">Daily</a>
            <a href="{% url 'planner:weekly' %}" class="btn active">Weekly</a>
            <a href="{% url 'planner:year' %}" class="btn">Year</a>
        </div>
    </div>

    <!-- Weekly Grid -->
    <div id="weekly-grid"
         hx-get="{% url 'planner:api_weekly_grid' %}"
         hx-trigger="load, refresh from:body"
         hx-swap="innerHTML">
        <!-- HTMX loads grid content -->
        <div class="loading">Loading...</div>
    </div>

    <!-- Stats -->
    <div id="weekly-stats"
         hx-get="{% url 'planner:api_weekly_stats' %}"
         hx-trigger="load, refresh from:body">
    </div>

    <!-- Legend -->
    <div class="weekly-legend">
        {% for category in categories %}
        <div class="legend-item">
            <div class="legend-color" style="background: {{ category.color }}"></div>
            <span>{{ category.label }}</span>
        </div>
        {% endfor %}
    </div>

</div>
{% endblock %}

{% block extra_js %}
<script src="{% static 'js/chart.min.js' %}"></script>
<script>
function weeklyView() {
    return {
        openActivityModal(activityId, weekday, dayType, scope) {
            const url = new URL("{% url 'planner:api_activity_modal' %}", window.location.origin);
            if (activityId) url.searchParams.set('id', activityId);
            url.searchParams.set('weekday', weekday);
            url.searchParams.set('day_type', dayType);
            url.searchParams.set('scope', scope);

            htmx.ajax('GET', url.toString(), {target: '#modal-content'})
                .then(() => this.$dispatch('modal-open'));
        },

        closeModal() {
            this.$dispatch('modal-close');
        },

        addActivity(event, weekday, dayType) {
            // Calculate time from click position
            const rect = event.currentTarget.getBoundingClientRect();
            const clickY = event.clientY - rect.top;
            const minutes = Math.floor(clickY / 40) * 60 + (clickY % 40 >= 20 ? 30 : 0);
            const startTime = this.formatTime(minutes);

            const url = new URL("{% url 'planner:api_activity_modal' %}", window.location.origin);
            url.searchParams.set('weekday', weekday);
            url.searchParams.set('day_type', dayType);
            url.searchParams.set('start_time', startTime);
            url.searchParams.set('scope', 'weekday');

            htmx.ajax('GET', url.toString(), {target: '#modal-content'})
                .then(() => this.$dispatch('modal-open'));
        },

        formatTime(minutes) {
            const h = Math.floor(minutes / 60);
            const m = minutes % 60;
            return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}`;
        }
    }
}
</script>
{% endblock %}
```

### 6.3 Weekly Grid Partial (apps/planner/templates/planner/partials/_weekly_grid.html)

```html
{% load planner_tags %}

<div class="weekly-grid">
    <!-- Time column -->
    <div class="time-column">
        <div class="time-column-header">Time</div>
        {% for hour in "0123456789"|make_list %}{% for h in "0123456789"|make_list %}
        {% with hour_num=hour|add:h %}
        {% if hour_num|length == 1 or hour_num|first != "0" %}
        {% if hour_num|add:"0" <= "23" %}
        <div class="time-slot">{{ hour_num|stringformat:"02d" }}:00</div>
        {% endif %}
        {% endif %}
        {% endwith %}
        {% endfor %}{% endfor %}
        {% for hour in 24|times %}
        <div class="time-slot">{{ hour|stringformat:"02d" }}:00</div>
        {% endfor %}
    </div>

    <!-- Day columns -->
    {% for day in weekdays %}
    <div class="day-column {% if day.is_weekend %}weekend{% endif %}"
         x-data
         @click="if (!$event.target.closest('.weekly-activity')) addActivity($event, '{{ day.name }}', '{{ day.day_type.name }}')">

        <div class="day-column-header">
            <div class="day-name">
                {{ day.name }}
                {% if day.has_custom %}
                <span class="custom-badge">Custom</span>
                {% endif %}
            </div>
            <select class="day-type-select"
                    hx-post="{% url 'planner:api_set_weekday_type' day.name %}"
                    hx-trigger="change"
                    hx-target="#weekly-grid"
                    name="day_type">
                {% for dt in day_types %}
                <option value="{{ dt.name }}" {% if dt.name == day.day_type.name %}selected{% endif %}>
                    {{ dt.name }}
                </option>
                {% endfor %}
            </select>
        </div>

        <div class="day-slots">
            <!-- Clickable background -->
            <div class="day-slots-bg"></div>

            <!-- Hour lines -->
            {% for hour in 24|times %}
            <div class="hour-line" style="top: {{ hour|multiply:40 }}px;"></div>
            <div class="half-hour-line" style="top: {{ hour|multiply:40|add:20 }}px;"></div>
            {% endfor %}

            <!-- Activities -->
            {% for activity in day.activities %}
            {% with segments=activity|get_segments %}
            {% for segment in segments %}
            <div class="weekly-activity {% if day.has_custom %}has-custom{% endif %}"
                 style="top: {{ segment.top }}px; height: {{ segment.height }}px; background: {{ activity.category.color }};"
                 @click.stop="openActivityModal('{{ activity.id }}', '{{ day.name }}', '{{ day.day_type.name }}', '{% if day.has_custom %}weekday{% else %}template{% endif %}')">
                <div class="weekly-activity-name">{{ activity.name }}</div>
                {% if segment.height > 25 %}
                <div class="weekly-activity-time">{{ activity.start_time|time:"H:i" }} - {{ activity.end_time|time:"H:i" }}</div>
                {% endif %}
            </div>
            {% endfor %}
            {% endwith %}
            {% endfor %}
        </div>
    </div>
    {% endfor %}
</div>
```

### 6.4 Activity Modal Partial (apps/planner/templates/planner/partials/_activity_modal.html)

```html
{% load static %}

<div class="weekly-edit-modal">
    <div class="weekly-edit-header">
        <span class="weekly-edit-activity-badge"
              style="background: {{ form.instance.category.color|default:'#e74c3c' }}">
            {% if is_new %}New Activity{% else %}{{ form.instance.category.label }}{% endif %}
        </span>
        <button class="weekly-edit-close" @click="closeModal()">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M18 6L6 18M6 6l12 12"/>
            </svg>
        </button>
    </div>

    <form hx-post="{% if is_new %}{% url 'planner:api_activity_create' %}{% else %}{% url 'planner:api_activity_update' activity_id %}{% endif %}"
          hx-target="#weekly-grid"
          hx-on::after-request="if(event.detail.successful) $dispatch('modal-close'); htmx.trigger('body', 'refresh');">

        {% csrf_token %}
        <input type="hidden" name="weekday" value="{{ weekday }}">
        <input type="hidden" name="day_type" value="{{ day_type }}">

        <div class="weekly-edit-form-row">
            <div class="weekly-edit-form-group weekly-edit-form-grow">
                <label for="id_name">Activity</label>
                {{ form.name }}
            </div>
        </div>

        <div class="weekly-edit-form-row">
            <div class="weekly-edit-form-group">
                <label for="id_start_time">Start</label>
                {{ form.start_time }}
            </div>
            <div class="weekly-edit-form-group">
                <label for="id_end_time">End</label>
                {{ form.end_time }}
            </div>
            <div class="weekly-edit-form-group">
                <label for="id_category">Category</label>
                {{ form.category }}
            </div>
            <div class="weekly-edit-form-group">
                <label for="id_color">Color</label>
                {{ form.color }}
            </div>
        </div>

        <div class="weekly-edit-section-title">Apply changes to:</div>

        <div class="weekly-edit-scope-options">
            <label class="weekly-edit-scope-option">
                <input type="radio" name="scope" value="weekday" {% if scope == 'weekday' %}checked{% endif %}>
                <span class="weekly-edit-scope-radio"></span>
                <div class="weekly-edit-scope-content">
                    <div class="weekly-edit-scope-title">All {{ weekday }}s</div>
                    <div class="weekly-edit-scope-desc">Only this weekday</div>
                </div>
            </label>
            <label class="weekly-edit-scope-option">
                <input type="radio" name="scope" value="template" {% if scope == 'template' %}checked{% endif %}>
                <span class="weekly-edit-scope-radio"></span>
                <div class="weekly-edit-scope-content">
                    <div class="weekly-edit-scope-title">{{ day_type }} template</div>
                    <div class="weekly-edit-scope-desc">All days using this template</div>
                </div>
            </label>
        </div>

        <div class="weekly-edit-actions">
            {% if not is_new %}
            <button type="button"
                    class="weekly-edit-btn weekly-edit-btn-delete"
                    hx-delete="{% url 'planner:api_activity_delete' activity_id %}"
                    hx-target="#weekly-grid"
                    hx-confirm="Delete this activity?"
                    hx-on::after-request="$dispatch('modal-close'); htmx.trigger('body', 'refresh');">
                <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M3 6h18M19 6v14a2 2 0 01-2 2H7a2 2 0 01-2-2V6m3 0V4a2 2 0 012-2h4a2 2 0 012 2v2"/>
                </svg>
                Delete
            </button>
            {% else %}
            <div></div>
            {% endif %}

            <div class="weekly-edit-actions-right">
                <button type="button" class="weekly-edit-btn weekly-edit-btn-cancel" @click="closeModal()">
                    Cancel
                </button>
                <button type="submit" class="weekly-edit-btn weekly-edit-btn-save">
                    Save
                </button>
            </div>
        </div>
    </form>
</div>
```

---

## 7. Authentication Setup

### 7.1 Install Dependencies

```bash
pip install django-allauth
```

### 7.2 Settings (config/settings/base.py)

```python
INSTALLED_APPS = [
    # ...
    'django.contrib.sites',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount.providers.apple',
]

SITE_ID = 1

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

# Allauth settings
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_AUTHENTICATION_METHOD = 'email'
ACCOUNT_EMAIL_VERIFICATION = 'optional'  # or 'mandatory' for production
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'

# Social auth providers (configure in admin)
SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'SCOPE': ['profile', 'email'],
        'AUTH_PARAMS': {'access_type': 'online'},
    },
}
```

---

## 8. Deployment

### 8.1 Recommended: Railway

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login and init
railway login
railway init

# Add PostgreSQL
railway add -p postgresql

# Deploy
railway up
```

### 8.2 Production Settings (config/settings/production.py)

```python
from .base import *
import os

DEBUG = False
ALLOWED_HOSTS = [os.environ.get('RAILWAY_STATIC_URL', '').replace('https://', '')]

# Database
import dj_database_url
DATABASES = {
    'default': dj_database_url.config(default=os.environ.get('DATABASE_URL'))
}

# Security
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000

# Static files (whitenoise)
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

### 8.3 Requirements.txt

```
django>=4.2
django-allauth>=0.54
psycopg2-binary>=2.9
dj-database-url>=2.0
whitenoise>=6.5
gunicorn>=21.0
python-dotenv>=1.0
```

---

## 9. Migration Strategy

### Phase 1: Setup (Week 1)
- [ ] Create Django project structure
- [ ] Set up models
- [ ] Create migrations
- [ ] Set up authentication
- [ ] Deploy empty app to Railway

### Phase 2: Core Views (Week 2)
- [ ] Convert Daily view
- [ ] Convert Weekly view
- [ ] Convert Year view
- [ ] HTMX activity modal

### Phase 3: Features (Week 3)
- [ ] Settings page
- [ ] Export/Import
- [ ] Onboarding wizard
- [ ] Category management

### Phase 4: Polish (Week 4)
- [ ] Error handling
- [ ] Loading states
- [ ] Mobile responsive fixes
- [ ] Testing

---

## 10. File Migration Map

| Current (V2.html) | Django Location |
|-------------------|-----------------|
| CSS (inline) | `static/css/main.css`, `weekly.css`, etc. |
| Clock rendering JS | `static/js/clock.js` |
| Chart.js code | `static/js/charts.js` |
| Activity modal | `templates/planner/partials/_activity_modal.html` |
| Weekly grid | `templates/planner/partials/_weekly_grid.html` |
| State/localStorage | Database models + Django sessions |
| Categories | `Category` model |
| Day types | `DayType` model |
| Activities | `Activity` + `WeekdayActivity` models |
| Day overrides | `DayOverride` model |

---

## 11. Strategic Planning Module (New Feature)

### 11.1 Concept: Purpose-to-Action Hierarchy

Transform the app from a time planner into a **life management system** by linking daily activities to higher-level purpose:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        VISION / MISSION / VALUES                        │
│                            (Why I exist)                                │
│                                                                         │
│  Vision:  "Become a recognized expert in AI and build impactful tools" │
│  Mission: "Help people make better use of their time through software" │
│  Values:  [Excellence] [Family First] [Continuous Learning] [Health]   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          STRATEGIC PILLARS                              │
│           (Where I choose to play & How I will win)                     │
│                                                                         │
│  1. "Build a SaaS product and launch it"                               │
│  2. "Maintain work-life balance (max 45h/week work)"                   │
│  3. "Improve health through consistent exercise"                        │
│  4. "Spend quality time with family every weekend"                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      PORTFOLIOS / LIFE AREAS                            │
│                   (Buckets that group related work)                     │
│                                                                         │
│  [Career]        [Health]        [Family]        [Personal Growth]     │
│   └─ Pillar 1     └─ Pillar 3     └─ Pillar 4     └─ Pillar 2          │
│   └─ Pillar 2                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         PROJECTS / GOALS                                │
│                    (Specific initiatives with deadlines)                │
│                                                                         │
│  Career Portfolio:                                                      │
│   ├─ Project: "Launch Daily Planner SaaS" (Q1 2026)                    │
│   │    └─ Goal: "Get 100 paying users by March"                        │
│   └─ Project: "Learn Django" (ongoing)                                 │
│        └─ Goal: "Complete tutorial by Jan 15"                          │
│                                                                         │
│  Health Portfolio:                                                      │
│   └─ Project: "Gym routine" (ongoing)                                  │
│        └─ Goal: "Exercise 3x per week"                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            ACTIVITIES                                   │
│                      (Daily time blocks) ← Already built                │
│                                                                         │
│  Monday 08:00-12:00: "Deep Work - Planner Feature"                     │
│       └─ Linked to: Project "Launch Daily Planner SaaS"                │
│       └─ Category: Work                                                 │
│       └─ Contributes to: Pillar 1, Portfolio: Career                   │
│                                                                         │
│  Monday 18:00-19:00: "Gym"                                             │
│       └─ Linked to: Project "Gym routine"                              │
│       └─ Category: Myself                                              │
│       └─ Contributes to: Pillar 3, Portfolio: Health                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Data Models

```python
# apps/strategy/models.py

from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()


class PersonalFoundation(models.Model):
    """User's vision, mission, and core values - the 'Why'"""
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='foundation')

    vision = models.TextField(
        blank=True,
        help_text="Long-term vision: What does your ideal future look like?"
    )
    mission = models.TextField(
        blank=True,
        help_text="Personal mission: What is your purpose? How do you create value?"
    )

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"Foundation for {self.user.username}"


class Value(models.Model):
    """Core values that guide decisions"""
    foundation = models.ForeignKey(
        PersonalFoundation,
        on_delete=models.CASCADE,
        related_name='values'
    )
    name = models.CharField(max_length=50)  # "Excellence", "Family", "Health"
    description = models.TextField(blank=True)
    icon = models.CharField(max_length=50, blank=True)  # Icon name or emoji
    sort_order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['sort_order', 'name']

    def __str__(self):
        return self.name


class StrategicPillar(models.Model):
    """
    Strategic pillars - based on 'Playing to Win' framework by A.G. Lafley & Roger Martin.

    Each pillar answers:
    - WHERE TO PLAY: Which arena/battlefield have you chosen?
    - HOW TO WIN: What is your competitive advantage in that arena?

    See also: Good Strategy Bad Strategy, Blue Ocean Strategy, Competitive Strategy
    """
    TIMEFRAME_CHOICES = [
        ('year', 'This Year'),
        ('quarter', 'This Quarter'),
        ('custom', 'Custom Period'),
    ]

    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='strategic_pillars')

    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)

    # Playing to Win framework fields
    where_to_play = models.TextField(
        blank=True,
        help_text="Which arena/market/battlefield have you chosen to compete in?"
    )
    how_to_win = models.TextField(
        blank=True,
        help_text="What is your competitive advantage? How will you win in this arena?"
    )

    timeframe = models.CharField(max_length=10, choices=TIMEFRAME_CHOICES, default='year')
    start_date = models.DateField(null=True, blank=True)
    end_date = models.DateField(null=True, blank=True)

    # Link to values this pillar supports
    values = models.ManyToManyField(Value, blank=True, related_name='pillars')

    is_active = models.BooleanField(default=True)
    sort_order = models.PositiveIntegerField(default=0)

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['sort_order', '-created_at']

    def __str__(self):
        return self.title


class Portfolio(models.Model):
    """Life areas / portfolios that group related work"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='portfolios')

    name = models.CharField(max_length=100)  # "Career", "Health", "Family"
    description = models.TextField(blank=True)
    color = models.CharField(max_length=7, default='#3498db')  # Hex color
    icon = models.CharField(max_length=50, blank=True)

    # Link to strategic pillars this portfolio supports
    pillars = models.ManyToManyField(
        StrategicPillar,
        blank=True,
        related_name='portfolios'
    )

    # Link to category (for automatic activity association)
    default_category = models.ForeignKey(
        'planner.Category',
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        help_text="Activities with this category auto-link to this portfolio"
    )

    is_active = models.BooleanField(default=True)
    sort_order = models.PositiveIntegerField(default=0)

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['sort_order', 'name']

    def __str__(self):
        return self.name


class Project(models.Model):
    """Specific initiatives within a portfolio"""
    STATUS_CHOICES = [
        ('planning', 'Planning'),
        ('active', 'Active'),
        ('on_hold', 'On Hold'),
        ('completed', 'Completed'),
        ('cancelled', 'Cancelled'),
    ]

    portfolio = models.ForeignKey(
        Portfolio,
        on_delete=models.CASCADE,
        related_name='projects'
    )

    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)

    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='planning')

    start_date = models.DateField(null=True, blank=True)
    target_end_date = models.DateField(null=True, blank=True)
    actual_end_date = models.DateField(null=True, blank=True)

    # Progress tracking
    progress_percent = models.PositiveIntegerField(default=0)

    # Link to strategic pillars
    pillars = models.ManyToManyField(
        StrategicPillar,
        blank=True,
        related_name='projects'
    )

    color = models.CharField(max_length=7, blank=True)  # Override portfolio color

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return f"{self.portfolio.name} / {self.name}"

    @property
    def is_overdue(self):
        from django.utils import timezone
        if self.target_end_date and self.status == 'active':
            return timezone.now().date() > self.target_end_date
        return False


class Goal(models.Model):
    """Measurable goals within a project"""
    TYPE_CHOICES = [
        ('outcome', 'Outcome Goal'),  # "Get 100 users"
        ('habit', 'Habit Goal'),       # "Exercise 3x/week"
        ('milestone', 'Milestone'),    # "Complete phase 1"
    ]

    project = models.ForeignKey(
        Project,
        on_delete=models.CASCADE,
        related_name='goals'
    )

    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)

    goal_type = models.CharField(max_length=20, choices=TYPE_CHOICES, default='outcome')

    # For measurable goals
    target_value = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    current_value = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    unit = models.CharField(max_length=50, blank=True)  # "users", "hours", "kg"

    # For habit goals
    frequency = models.CharField(max_length=50, blank=True)  # "3x per week"

    target_date = models.DateField(null=True, blank=True)
    is_completed = models.BooleanField(default=False)
    completed_at = models.DateTimeField(null=True, blank=True)

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['target_date', '-created_at']

    def __str__(self):
        return self.title

    @property
    def progress_percent(self):
        if self.target_value and self.current_value:
            return min(100, int((self.current_value / self.target_value) * 100))
        return 0 if not self.is_completed else 100


class Roadmap(models.Model):
    """High-level roadmap / timeline view"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='roadmaps')

    name = models.CharField(max_length=200)  # "2026 Roadmap"
    year = models.PositiveIntegerField()

    description = models.TextField(blank=True)

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = ['user', 'year']

    def __str__(self):
        return self.name


class RoadmapMilestone(models.Model):
    """Milestones on the roadmap"""
    QUARTER_CHOICES = [
        (1, 'Q1'),
        (2, 'Q2'),
        (3, 'Q3'),
        (4, 'Q4'),
    ]

    roadmap = models.ForeignKey(
        Roadmap,
        on_delete=models.CASCADE,
        related_name='milestones'
    )

    title = models.CharField(max_length=200)
    quarter = models.PositiveIntegerField(choices=QUARTER_CHOICES)

    # Link to project/goal
    project = models.ForeignKey(
        Project,
        on_delete=models.SET_NULL,
        null=True,
        blank=True
    )
    goal = models.ForeignKey(
        Goal,
        on_delete=models.SET_NULL,
        null=True,
        blank=True
    )

    is_completed = models.BooleanField(default=False)

    sort_order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['quarter', 'sort_order']

    def __str__(self):
        return f"Q{self.quarter}: {self.title}"
```

### 11.3 Link Activities to Projects

Update the Activity model to link to projects:

```python
# Update apps/planner/models.py

class Activity(models.Model):
    template = models.ForeignKey(DayTemplate, on_delete=models.CASCADE, related_name='activities')
    name = models.CharField(max_length=100)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    start_time = models.TimeField()
    end_time = models.TimeField()
    color = models.CharField(max_length=7, blank=True)
    sort_order = models.PositiveIntegerField(default=0)

    # NEW: Link to project (optional)
    project = models.ForeignKey(
        'strategy.Project',
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='activities',
        help_text="Link this activity to a project for tracking"
    )

    # ... rest of model


class WeekdayActivity(models.Model):
    # ... existing fields ...

    # NEW: Link to project (optional)
    project = models.ForeignKey(
        'strategy.Project',
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='weekday_activities'
    )
```

### 11.4 Updated Project Structure

```
dailyplanner/
├── apps/
│   ├── accounts/           # User management
│   ├── planner/            # Time planning (existing)
│   │   ├── models.py       # Activity, DayTemplate, etc.
│   │   └── ...
│   │
│   └── strategy/           # NEW: Strategic planning
│       ├── __init__.py
│       ├── models.py       # Foundation, Strategy, Portfolio, Project, Goal
│       ├── views.py        # Strategy views
│       ├── urls.py
│       ├── forms.py
│       ├── api.py          # HTMX endpoints
│       ├── services.py     # Business logic (alignment calculations)
│       └── templates/
│           └── strategy/
│               ├── dashboard.html      # Strategy overview
│               ├── foundation.html     # Vision/Mission/Values editor
│               ├── pillars.html        # Strategic pillars list
│               ├── portfolios.html     # Portfolio management
│               ├── projects.html       # Projects list
│               ├── project_detail.html # Single project view
│               ├── goals.html          # Goals tracking
│               ├── roadmap.html        # Timeline view
│               └── partials/
│                   ├── _pillar_card.html
│                   ├── _portfolio_card.html
│                   ├── _project_card.html
│                   ├── _goal_item.html
│                   ├── _roadmap_quarter.html
│                   └── _alignment_chart.html
```

### 11.5 URL Routes for Strategy Module

```python
# apps/strategy/urls.py

from django.urls import path
from . import views, api

app_name = 'strategy'

urlpatterns = [
    # Main views
    path('', views.StrategyDashboardView.as_view(), name='dashboard'),
    path('foundation/', views.FoundationView.as_view(), name='foundation'),
    path('pillars/', views.PillarsView.as_view(), name='pillars'),
    path('portfolios/', views.PortfoliosView.as_view(), name='portfolios'),
    path('projects/', views.ProjectsView.as_view(), name='projects'),
    path('projects/<int:pk>/', views.ProjectDetailView.as_view(), name='project_detail'),
    path('goals/', views.GoalsView.as_view(), name='goals'),
    path('roadmap/', views.RoadmapView.as_view(), name='roadmap'),
    path('roadmap/<int:year>/', views.RoadmapView.as_view(), name='roadmap_year'),

    # HTMX API endpoints
    path('api/foundation/save/', api.save_foundation, name='api_save_foundation'),
    path('api/value/create/', api.create_value, name='api_create_value'),
    path('api/value/<int:pk>/delete/', api.delete_value, name='api_delete_value'),

    path('api/pillar/create/', api.create_pillar, name='api_create_pillar'),
    path('api/pillar/<int:pk>/update/', api.update_pillar, name='api_update_pillar'),
    path('api/pillar/<int:pk>/delete/', api.delete_pillar, name='api_delete_pillar'),

    path('api/portfolio/create/', api.create_portfolio, name='api_create_portfolio'),
    path('api/portfolio/<int:pk>/update/', api.update_portfolio, name='api_update_portfolio'),
    path('api/portfolio/<int:pk>/delete/', api.delete_portfolio, name='api_delete_portfolio'),

    path('api/project/create/', api.create_project, name='api_create_project'),
    path('api/project/<int:pk>/update/', api.update_project, name='api_update_project'),
    path('api/project/<int:pk>/delete/', api.delete_project, name='api_delete_project'),
    path('api/project/<int:pk>/progress/', api.update_project_progress, name='api_project_progress'),

    path('api/goal/create/', api.create_goal, name='api_create_goal'),
    path('api/goal/<int:pk>/update/', api.update_goal, name='api_update_goal'),
    path('api/goal/<int:pk>/toggle/', api.toggle_goal, name='api_toggle_goal'),
    path('api/goal/<int:pk>/delete/', api.delete_goal, name='api_delete_goal'),

    path('api/roadmap/milestone/create/', api.create_milestone, name='api_create_milestone'),
    path('api/roadmap/milestone/<int:pk>/toggle/', api.toggle_milestone, name='api_toggle_milestone'),

    # Analytics
    path('api/alignment/', api.get_alignment_data, name='api_alignment'),
    path('api/time-by-portfolio/', api.get_time_by_portfolio, name='api_time_by_portfolio'),
    path('api/time-by-pillar/', api.get_time_by_pillar, name='api_time_by_pillar'),
]
```

### 11.6 Strategy Service Layer

```python
# apps/strategy/services.py

from datetime import timedelta
from django.db.models import Sum
from django.utils import timezone
from .models import Portfolio, Project, StrategicPillar
from apps.planner.models import Activity, WeekdayActivity


class AlignmentAnalyzer:
    """
    Analyzes how well daily activities align with strategic priorities.
    """

    def __init__(self, user):
        self.user = user

    def get_weekly_time_by_portfolio(self):
        """
        Calculate total weekly time spent on each portfolio.
        Returns dict: {portfolio_id: minutes}
        """
        portfolios = Portfolio.objects.filter(user=self.user, is_active=True)
        result = {}

        for portfolio in portfolios:
            # Get time from activities linked to this portfolio's projects
            project_ids = portfolio.projects.values_list('id', flat=True)

            # Sum from template activities
            template_time = Activity.objects.filter(
                project_id__in=project_ids
            ).aggregate(
                total=Sum('duration_minutes')
            )['total'] or 0

            # Sum from weekday activities
            weekday_time = WeekdayActivity.objects.filter(
                project_id__in=project_ids
            ).aggregate(
                total=Sum('duration_minutes')
            )['total'] or 0

            # Also count activities by category if portfolio has default_category
            if portfolio.default_category:
                category_time = self._get_time_by_category(portfolio.default_category)
                result[portfolio.id] = template_time + weekday_time + category_time
            else:
                result[portfolio.id] = template_time + weekday_time

        return result

    def get_weekly_time_by_pillar(self):
        """
        Calculate weekly time allocated to each strategic pillar.
        """
        pillars = StrategicPillar.objects.filter(user=self.user, is_active=True)
        result = {}

        for pillar in pillars:
            total_time = 0

            # Get all projects linked to this pillar
            for project in pillar.projects.all():
                total_time += self._get_project_weekly_time(project)

            # Get all portfolios linked to this pillar
            for portfolio in pillar.portfolios.all():
                total_time += self._get_portfolio_weekly_time(portfolio)

            result[pillar.id] = total_time

        return result

    def get_alignment_score(self):
        """
        Calculate overall alignment score (0-100).
        Measures how much of your planned time goes to strategic pillars.
        """
        total_weekly_time = self._get_total_weekly_planned_time()
        pillar_time = sum(self.get_weekly_time_by_pillar().values())

        if total_weekly_time == 0:
            return 0

        return min(100, int((pillar_time / total_weekly_time) * 100))

    def get_portfolio_balance(self):
        """
        Analyze balance across portfolios.
        Returns insights about over/under-invested areas.
        """
        time_by_portfolio = self.get_weekly_time_by_portfolio()
        total_time = sum(time_by_portfolio.values())

        if total_time == 0:
            return []

        result = []
        for portfolio in Portfolio.objects.filter(user=self.user, is_active=True):
            time = time_by_portfolio.get(portfolio.id, 0)
            percent = int((time / total_time) * 100) if total_time > 0 else 0

            result.append({
                'portfolio': portfolio,
                'minutes': time,
                'hours': time / 60,
                'percent': percent,
            })

        return sorted(result, key=lambda x: -x['minutes'])

    def _get_total_weekly_planned_time(self):
        """Get total weekly planned time in minutes"""
        # Sum all activities across the week
        # This would use the planner service
        from apps.planner.services import ActivityResolver
        resolver = ActivityResolver(self.user)

        total = 0
        for day_data in resolver.get_weekly_data():
            for activity in day_data['activities']:
                total += activity.duration_minutes

        return total

    def _get_project_weekly_time(self, project):
        """Get weekly time for a specific project"""
        template_time = Activity.objects.filter(project=project).aggregate(
            total=Sum('duration_minutes')
        )['total'] or 0

        weekday_time = WeekdayActivity.objects.filter(project=project).aggregate(
            total=Sum('duration_minutes')
        )['total'] or 0

        return template_time + weekday_time

    def _get_portfolio_weekly_time(self, portfolio):
        """Get weekly time for a portfolio"""
        total = 0
        for project in portfolio.projects.filter(status='active'):
            total += self._get_project_weekly_time(project)
        return total

    def _get_time_by_category(self, category):
        """Get weekly time for activities with this category (not linked to project)"""
        # Only count activities without explicit project link
        template_time = Activity.objects.filter(
            category=category,
            project__isnull=True
        ).aggregate(total=Sum('duration_minutes'))['total'] or 0

        weekday_time = WeekdayActivity.objects.filter(
            category=category,
            project__isnull=True
        ).aggregate(total=Sum('duration_minutes'))['total'] or 0

        return template_time + weekday_time
```

### 11.7 UI Pages

#### Strategy Dashboard
```
┌─────────────────────────────────────────────────────────────────────────┐
│  Strategy Dashboard                              [Foundation] [Roadmap] │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │ Alignment    │  │ Active       │  │ Goals        │                  │
│  │ Score        │  │ Projects     │  │ Progress     │                  │
│  │    73%       │  │     5        │  │   12/18      │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                         │
│  STRATEGIC PILLARS (2026)                                    [+ Add]   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. Build a SaaS product          [Career] [Personal Growth]     │   │
│  │    Where: B2C productivity market | How: Visual time planning   │   │
│  │    └─ 3 projects, 2 goals                           32h/week    │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ 2. Maintain work-life balance    [Family] [Health]              │   │
│  │    Where: Home life | How: Strict boundaries + quality time     │   │
│  │    └─ 2 projects, 4 goals                           15h/week    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  PORTFOLIO BALANCE                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Career      ████████████████████░░░░░░░  45%   (32h)            │   │
│  │ Health      ██████░░░░░░░░░░░░░░░░░░░░░  12%   (8h)             │   │
│  │ Family      █████████░░░░░░░░░░░░░░░░░░  18%   (12h)            │   │
│  │ Personal    ███████░░░░░░░░░░░░░░░░░░░░  15%   (10h)            │   │
│  │ Unallocated ████░░░░░░░░░░░░░░░░░░░░░░░  10%   (7h)             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Foundation Page
```
┌─────────────────────────────────────────────────────────────────────────┐
│  Vision, Mission & Values                                     [Save]   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VISION                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ What does your ideal future look like in 5-10 years?            │   │
│  │                                                                  │   │
│  │ Become a recognized expert in AI-powered productivity tools,    │   │
│  │ running a profitable SaaS business while maintaining a healthy  │   │
│  │ work-life balance and being present for my family.              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  MISSION                                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ What is your purpose? How do you create value?                  │   │
│  │                                                                  │   │
│  │ Help people make better use of their time through intuitive     │   │
│  │ software that transforms planning from a chore into insight.    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  CORE VALUES                                                   [+ Add] │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐          │
│  │ Excellence │ │ Family     │ │ Learning   │ │ Health     │          │
│  │            │ │ First      │ │            │ │            │          │
│  │ [Edit] [x] │ │ [Edit] [x] │ │ [Edit] [x] │ │ [Edit] [x] │          │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Roadmap Page
```
┌─────────────────────────────────────────────────────────────────────────┐
│  2026 Roadmap                           [< 2025]  [2026]  [2027 >]     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│      Q1 (Jan-Mar)         Q2 (Apr-Jun)         Q3 (Jul-Sep)            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │ [x] Launch MVP  │  │ [ ] 500 users   │  │ [ ] Mobile app  │         │
│  │     Jan 31      │  │     Jun 15      │  │     Sep 30      │         │
│  │                 │  │                 │  │                 │         │
│  │ [ ] 100 users   │  │ [ ] Hire VA     │  │ [ ] $5k MRR     │         │
│  │     Mar 15      │  │     May 01      │  │                 │         │
│  │                 │  │                 │  │                 │         │
│  │ [x] Django      │  │ [ ] Marketing   │  │                 │         │
│  │     tutorial    │  │     campaign    │  │                 │         │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘         │
│                                                                         │
│                            Q4 (Oct-Dec)                                 │
│                        ┌─────────────────┐                              │
│                        │ [ ] 1000 users  │                              │
│                        │                 │                              │
│                        │ [ ] Team of 2   │                              │
│                        │                 │                              │
│                        └─────────────────┘                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.8 Integration with Weekly View

Update the activity modal to allow linking to projects:

```html
<!-- In _activity_modal.html -->
<div class="weekly-edit-form-row">
    <div class="weekly-edit-form-group weekly-edit-form-grow">
        <label for="weeklyEditProject">Link to Project (optional)</label>
        <select id="weeklyEditProject" name="project">
            <option value="">-- No project --</option>
            {% for portfolio in portfolios %}
            <optgroup label="{{ portfolio.name }}">
                {% for project in portfolio.projects.all %}
                <option value="{{ project.id }}"
                        {% if activity.project_id == project.id %}selected{% endif %}>
                    {{ project.name }}
                </option>
                {% endfor %}
            </optgroup>
            {% endfor %}
        </select>
    </div>
</div>
```

### 11.9 Migration Strategy Update

Add Phase 5 to the migration plan:

```
### Phase 5: Strategic Planning (Weeks 5-6)
- [ ] Create strategy app with models
- [ ] Foundation page (Vision/Mission/Values)
- [ ] Strategic Pillars CRUD (Where to Play / How to Win)
- [ ] Portfolio management
- [ ] Project & Goal tracking
- [ ] Roadmap view
- [ ] Link activities to projects
- [ ] Alignment analytics
- [ ] Strategy dashboard
```

### 11.10 Model Relationships Diagram (Updated)

```
User
 │
 ├── PersonalFoundation (1:1)
 │    └── Value (1:N)
 │
 ├── StrategicPillar (1:N)
 │    ├── linked to Values (M:N)
 │    ├── linked to Portfolios (M:N)
 │    └── linked to Projects (M:N)
 │
 ├── Portfolio (1:N)
 │    ├── linked to Pillars (M:N)
 │    ├── linked to Category (optional)
 │    └── Project (1:N)
 │         ├── linked to Pillars (M:N)
 │         ├── Goal (1:N)
 │         ├── Activity (1:N) ← from planner
 │         └── WeekdayActivity (1:N) ← from planner
 │
 ├── Roadmap (1:N by year)
 │    └── RoadmapMilestone (1:N)
 │         ├── linked to Project
 │         └── linked to Goal
 │
 └── [Planner models - existing]
      ├── Category
      ├── DayType → DayTemplate → Activity (+ project link)
      ├── WeekdayConfig → WeekdayActivity (+ project link)
      └── DayOverride
```

---

*Document Version: 1.2*
*Created: January 2026*
*Updated: January 2026 - Renamed Strategic Choices to Strategic Pillars (Playing to Win framework)*
