# PYTHON-PROJECT
# EMPLOYEE TIME TRACKER PROJECT USING DJANGO BY PYTHON
django-admin startproject employee_time_tracker
cd employee_time_tracker
python manage.py startapp tracker

#In tracker/models.py, define the models for Users, Tasks, and Projects
from django.db import models
from django.contrib.auth.models import User

class Project(models.Model):
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True)

   def __str__(self):
        return self.name

class Task(models.Model):
    CATEGORY_CHOICES = [
        ('Meeting', 'Meeting'),
        ('Training', 'Training'),
        ('Development', 'Development'),
        ('Other', 'Other'),
    ]
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    project = models.ForeignKey(Project, on_delete=models.CASCADE)
    date = models.DateField()
    start_time = models.TimeField()
    end_time = models.TimeField()
    duration = models.FloatField()  # Duration in hours
    category = models.CharField(max_length=50, choices=CATEGORY_CHOICES)
    description = models.TextField()

    def __str__(self):
        return f"{self.user.username} - {self.project.name} - {self.date}"

    def clean(self):
        if self.duration > 8:
            raise ValidationError('Duration cannot exceed 8 hours.')
        if Task.objects.filter(user=self.user, date=self.date, start_time=self.start_time).exists():
            raise ValidationError('Duplicate entry for the same date and time.')
#In tracker/forms.py, create forms for adding and editing tasks.
from django import forms
from .models import Task

class TaskForm(forms.ModelForm):
    class Meta:
        model = Task
        fields = ['project', 'date', 'start_time', 'end_time', 'duration', 'category', 'description']
#In tracker/views.py, define views for managing tasks and displaying charts
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required, user_passes_test
from .models import Task, Project
from .forms import TaskForm
from django.http import JsonResponse
from django.db.models import Sum
from datetime import datetime, timedelta

@login_required
def task_list(request):
    tasks = Task.objects.filter(user=request.user)
    return render(request, 'time_tracker/task_list.html', {'tasks': tasks})

@login_required
def add_task(request):
    if request.method == 'POST':
        form = TaskForm(request.POST)
        if form.is_valid():
            task = form.save(commit=False)
            task.user = request.user
            task.save()
            return redirect('task_list')
    else:
        form = TaskForm()
    return render(request, 'time_tracker/add_task.html', {'form': form})

@login_required
def edit_task(request, task_id):
    task = Task.objects.get(id=task_id)
    if request.method == 'POST':
        form = TaskForm(request.POST, instance=task)
        if form.is_valid():
            form.save()
            return redirect('task_list')
    else:
        form = TaskForm(instance=task)
    return render(request, 'time_tracker/edit_task.html', {'form': form})

@login_required
def delete_task(request, task_id):
    task = Task.objects.get(id=task_id)
    if request.method == 'POST':
        task.delete()
        return redirect('task_list')
    return render(request, 'time_tracker/delete_task.html', {'task': task})

@login_required
def generate_report(request):
    # Example of generating a daily report
    today = datetime.now().date()
    tasks = Task.objects.filter(user=request.user, date=today)
    total_duration = tasks.aggregate(Sum('duration'))['duration__sum']
    return JsonResponse({'total_duration': total_duration})
from django.urls import path
from . import views

urlpatterns = [
    path('tasks/', views.task_list, name='task_list'),
    path('tasks/add/', views.add_task, name='add_task'),
    path('tasks/edit/<int:task_id>/', views.edit_task, name='edit_task'),
    path('tasks/delete/<int:task_id>/', views.delete_task, name='delete_task'),
    path('report/', views.generate_report, name='generate_report'),
]

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('tracker/', include('tracker.urls')),
]

<!DOCTYPE html>
<html>
<head>
    <title>Task List</title>
</head>
<body>
    <h1>Task List</h1>
    <a href="{% url 'add_task' %}">Add New Task</a>
    <ul>
        {% for task in tasks %}
        <li>
            <a href="{% url 'task_detail' task.pk %}">{{ task.date }} - {{ task.category }}</a>
        </li>
        {% endfor %}
    </ul>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <title>Add Task</title>
</head>
<body>
    <h1>Add Task</h1>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Save</button>
    </form>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <title>Charts</title>
</head>
<body>
    <h1>Charts</h1>
    <img src="data:image/png;base64,{{ chart }}" />
</body>
</html>

python manage.py makemigrations
python manage.py migrate

python manage.py createsuperuser

python manage.py runserver



