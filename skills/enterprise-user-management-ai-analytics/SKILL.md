---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "create user management dashboard with AI analytics"
  - "implement AI-powered ticket classification"
  - "build task tracking system with anomaly detection"
  - "configure user management with risk prediction"
  - "deploy enterprise admin dashboard with ML features"
  - "integrate AI analytics for user behavior monitoring"
  - "create kanban board with burnout detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform combining React frontend, Node.js backend, and FastAPI ML service for intelligent user administration, task tracking, support ticket management, and AI-driven analytics including risk detection, anomaly detection, and burnout analysis.

## What It Does

This system provides:
- **User Management**: JWT-authenticated CRUD operations for users with role-based access control
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, and user monitoring
- **User Dashboard**: Personal task overview, performance insights, and notifications

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or remotely

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// Login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};

// Protected API call with JWT
const getProtectedData = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users/profile', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};
```

### User Management (Admin)

```javascript
// Get all users (admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/admin/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update user role
const updateUserRole = async (userId, newRole) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/admin/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ role: newRole })
  });
  return response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/admin/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority || 'medium',
      status: 'todo',
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Get user tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update task status (Kanban)
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus }) // 'todo', 'in-progress', 'done'
  });
  return response.json();
};

// Track time on task
const startTimeTracking = async (taskId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time-start`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

const stopTimeTracking = async (taskId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time-stop`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority || 'medium'
    })
  });
  return response.json();
};

// Get user tickets
const getUserTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update ticket status (admin)
const updateTicketStatus = async (ticketId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'open', 'in-progress', 'resolved', 'closed'
  });
  return response.json();
};
```

## ML Service API Usage

### AI Ticket Classification

```javascript
// Classify ticket automatically
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "User unable to login"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', department: 'IT' }
  return result;
};
```

### Risk Prediction

```javascript
// Predict user risk based on behavior
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      failed_logins: 3,
      after_hours_activity: 15,
      data_access_volume: 8500,
      privilege_escalation_attempts: 2
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.78, risk_level: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalous user behavior
const detectAnomalies = async (userBehavior) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userBehavior.userId,
      login_time: userBehavior.loginTime,
      location: userBehavior.location,
      device_type: userBehavior.deviceType,
      actions_count: userBehavior.actionsCount
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.89, details: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// Detect employee burnout risk
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      tasks_completed: 45,
      average_hours_per_day: 11.5,
      missed_deadlines: 3,
      weekend_work_frequency: 4,
      time_since_vacation: 180 // days
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.85, recommendations: [...] }
  return result;
};
```

### Project Delay Prediction

```javascript
// Predict project delay probability
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      average_task_completion_time: projectData.avgCompletionTime
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.72, estimated_delay_days: 5, recommendations: [...] }
  return result;
};
```

## React Frontend Patterns

### User Dashboard Component

```javascript
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [tickets, setTickets] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };

      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tickets/my-tickets`, config)
      ]);

      // Group tasks by status
      const groupedTasks = {
        todo: tasksRes.data.filter(t => t.status === 'todo'),
        inProgress: tasksRes.data.filter(t => t.status === 'in-progress'),
        done: tasksRes.data.filter(t => t.status === 'done')
      };

      setTasks(groupedTasks);
      setTickets(ticketsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchDashboardData(); // Refresh
    } catch (error) {
      console.error('Error updating task status:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      <div className="kanban-board">
        <div className="column">
          <h2>To Do ({tasks.todo.length})</h2>
          {tasks.todo.map(task => (
            <TaskCard
              key={task._id}
              task={task}
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>
        <div className="column">
          <h2>In Progress ({tasks.inProgress.length})</h2>
          {tasks.inProgress.map(task => (
            <TaskCard
              key={task._id}
              task={task}
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>
        <div className="column">
          <h2>Done ({tasks.done.length})</h2>
          {tasks.done.map(task => (
            <TaskCard
              key={task._id}
              task={task}
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>
      </div>
      <div className="tickets-section">
        <h2>My Tickets</h2>
        <TicketList tickets={tickets} />
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component with AI Integration

```javascript
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskAlerts, setRiskAlerts] = useState([]);
  const [burnoutAlerts, setBurnoutAlerts] = useState([]);
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };

      const usersRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/admin/users`,
        config
      );
      setUsers(usersRes.data);

      // Run AI analytics for each user
      const riskChecks = await Promise.all(
        usersRes.data.map(user =>
          axios.post(
            `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`,
            { user_id: user._id, ...user.behavior_metrics }
          )
        )
      );

      const highRiskUsers = riskChecks
        .map((res, idx) => ({ ...res.data, user: usersRes.data[idx] }))
        .filter(r => r.risk_level === 'high');

      setRiskAlerts(highRiskUsers);

      // Check burnout
      const burnoutChecks = await Promise.all(
        usersRes.data.map(user =>
          axios.post(
            `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`,
            { user_id: user._id, ...user.workload_metrics }
          )
        )
      );

      const burnoutRisk = burnoutChecks
        .map((res, idx) => ({ ...res.data, user: usersRes.data[idx] }))
        .filter(b => b.burnout_risk === 'high' || b.burnout_risk === 'medium');

      setBurnoutAlerts(burnoutRisk);

      // Organization-wide analytics
      const analyticsRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
        config
      );
      setAnalytics(analyticsRes.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>Admin Analytics Dashboard</h1>

      <div className="alert-section">
        <h2>Risk Alerts ({riskAlerts.length})</h2>
        {riskAlerts.map(alert => (
          <div key={alert.user._id} className="alert-card risk">
            <h3>{alert.user.name}</h3>
            <p>Risk Score: {(alert.risk_score * 100).toFixed(0)}%</p>
            <p>Factors: {alert.factors.join(', ')}</p>
          </div>
        ))}
      </div>

      <div className="alert-section">
        <h2>Burnout Alerts ({burnoutAlerts.length})</h2>
        {burnoutAlerts.map(alert => (
          <div key={alert.user._id} className="alert-card burnout">
            <h3>{alert.user.name}</h3>
            <p>Burnout Risk: {alert.burnout_risk}</p>
            <p>Recommendations: {alert.recommendations.join(', ')}</p>
          </div>
        ))}
      </div>

      {analytics && (
        <div className="org-stats">
          <h2>Organization Statistics</h2>
          <div className="stat-grid">
            <div className="stat">
              <h3>{analytics.total_users}</h3>
              <p>Total Users</p>
            </div>
            <div className="stat">
              <h3>{analytics.active_tasks}</h3>
              <p>Active Tasks</p>
            </div>
            <div className="stat">
              <h3>{analytics.open_tickets}</h3>
              <p>Open Tickets</p>
            </div>
            <div className="stat">
              <h3>{analytics.completion_rate}%</h3>
              <p>Task Completion Rate</p>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

export default AdminAnalytics;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [intervalId, setIntervalId] = useState(null);

  const startTracking = async () => {
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time-start`,
        {},
        { headers: { Authorization: `Bearer ${token}` } }
      );

      setIsTracking(true);
      const id = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
      setIntervalId(id);
    } catch (error) {
      console.error('Error starting time tracking:', error);
    }
  };

  const stopTracking = async () => {
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time-stop`,
        {},
        { headers: { Authorization: `Bearer ${token}` } }
      );

      clearInterval(intervalId);
      setIsTracking(false);
      setElapsedTime(0);
    } catch (error) {
      console.error('Error stopping time tracking:', error);
    }
  };

  useEffect(() => {
    return () => {
      if (intervalId) clearInterval(intervalId);
    };
  }, [intervalId]);

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      {!isTracking ? (
        <button onClick={startTracking} className="btn-start">
          Start Timer
        </button>
      ) : (
        <button onClick={stopTracking} className="btn-stop">
          Stop Timer
        </button>
      )}
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
# Or use MongoDB Atlas
# MONGODB_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/dbname

# Authentication
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional for notifications)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your_email@gmail.com
EMAIL_PASS=your_app_password
```

### ML Service Environment Variables

```env
# API
HOST=0.0.0.0
PORT=8000

# Models
MODEL_PATH=./models
ENABLE_MODEL_TRAINING=true

# Backend Integration
BACKEND_URL=http://localhost:5000

# Logging
LOG_LEVEL=INFO
```

### Frontend Environment Variables

```env
# API Endpoints
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# Feature Flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_TIME_TRACKING=true
```

## Common Patterns

### Middleware for JWT Authentication (Backend)

```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Axios Instance with Interceptors (Frontend)

```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Request interceptor to add token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor to handle errors
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Real-time Notifications Hook

```javascript
import { useEffect, useState } from 'react';
import axios from 'axios';

const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const fetchNotifications = async () => {
      try {
        const token = localStorage.getItem('token');
        const response = await axios.get(
          `${process.env.REACT_APP_API_URL}/api/notifications`,
          { headers: { Authorization: `Bearer ${token}` } }
        );
        setNotifications(response.data);
      } catch (error) {
        console.error('Error fetching notifications:', error);
      }
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s

    return () => clearInterval(interval);
  }, []);

  const markAsRead = async (notificationId) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/notifications/${notificationId}/read`,
        {},
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setNotifications(prev =>
        prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
      );
    } catch (error) {
      console.error('Error marking notification as read:', error);
    }
  };

  return { notifications, markAsRead };
};

export default useNotifications;
```

## Troubleshooting

### Issue: JWT Token Expired

**Solution**: Implement token refresh mechanism or handle 401 errors gracefully:

```javascript
// In axios interceptor
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const response = await axios.post('/api/auth/refresh', { refreshToken });
        localStorage.setItem('token', response.data.token);
        originalRequest.headers.Authorization = `Bearer ${response.data.token}`;
        return api(originalRequest);
      } catch (err) {
        localStorage.clear();
        window.location.href = '/login';
      }
    }
    return Promise.reject(error);
  }
);
```

### Issue: ML Service Not Responding

**Symptoms**: AI features fail, ticket classification doesn't work

**Solution**: Check ML service health endpoint:

```javascript
const checkMLServiceHealth = async () => {
  try {
    const response = await axios.get(`${process.env.REACT_APP_ML_API_URL}/health`);
    console.log('ML Service status:', response.data);
    return true;
  } catch (error) {
    console.error('ML Service is down:', error);
    // Fallback to non-AI features
    return false;
  }
};

// Use in components
const [mlAvailable, setMlAvailable] = useState(false);

useEffect(() => {
  checkMLServiceHealth().then(setMlAvailable);
}, []);
```

### Issue: MongoDB Connection Failed

**Solution**: Verify MongoDB URI and connection:

```javascript
// backend/db/connection.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Issue: CORS Errors in Development

**Solution**: Configure CORS in backend:

```javascript
const cors = require('cors');

app.use(cors({
  origin: process.env.NODE_ENV === 'production'
    ? 'https://your-frontend-domain.com'
    : 'http://localhost:3000',
  credentials: true
}));
```

### Issue: AI Models Not Trained

**Solution**: Initialize and train models on first run:

```python
# ml-service/main.py
from fastapi import FastAPI
import os
import pickle

app = FastAPI()

@app.on_event("startup")
async def startup_event():
    model_path = os.get
