---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, ticket routing, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "build user management system with task tracking"
  - "implement AI-powered ticket classification"
  - "create admin dashboard with user analytics"
  - "add burnout detection to user management"
  - "integrate predictive analytics for user tasks"
  - "develop kanban board with time tracking"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system that combines administrative tools, task tracking, and AI-powered analytics. It provides role-based access control, Kanban boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and project delay forecasting.

## Project Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User interface for admins and regular users
2. **Backend** (Node.js/Express) - REST API server with JWT authentication
3. **ML Service** (FastAPI + scikit-learn + River) - AI analytics engine

## Installation

### Prerequisites

- Node.js 16+ and npm
- Python 3.8+
- MongoDB instance

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
LOG_LEVEL=info
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Backend API Reference

### Authentication

**Login**
```javascript
// POST /api/auth/login
const response = await fetch('http://localhost:5000/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'user@example.com',
    password: 'password123'
  })
});

const { token, user } = await response.json();
// Store token for subsequent requests
localStorage.setItem('authToken', token);
```

**Register User**
```javascript
// POST /api/auth/register
const response = await fetch('http://localhost:5000/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com',
    password: 'securepass123',
    role: 'user' // or 'admin'
  })
});
```

### User Management (Admin Only)

**Get All Users**
```javascript
// GET /api/users
const response = await fetch('http://localhost:5000/api/users', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`
  }
});

const users = await response.json();
```

**Update User**
```javascript
// PUT /api/users/:id
const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'Updated Name',
    role: 'admin',
    status: 'active'
  })
});
```

**Delete User**
```javascript
// DELETE /api/users/:id
await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'DELETE',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`
  }
});
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
const response = await fetch('http://localhost:5000/api/tasks', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Implement feature X',
    description: 'Add new authentication method',
    assignedTo: 'userId123',
    priority: 'high',
    deadline: '2026-12-31',
    status: 'todo'
  })
});
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`
  }
});

const tasks = await response.json();
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: 'in-progress' // or 'done', 'todo'
  })
});
```

**Track Time**
```javascript
// POST /api/tasks/:id/time
await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    duration: 3600, // seconds
    action: 'start' // or 'stop', 'pause'
  })
});
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const response = await fetch('http://localhost:5000/api/tickets', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    subject: 'Login issue',
    description: 'Cannot access dashboard after password reset',
    category: 'technical',
    priority: 'medium'
  })
});
```

**Get User Tickets**
```javascript
// GET /api/tickets/user
const response = await fetch('http://localhost:5000/api/tickets/user', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`
  }
});

const tickets = await response.json();
```

## ML Service API Reference

### AI-Powered Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    subject: 'Cannot access account',
    description: 'Getting 403 error when trying to login',
    user_history: {
      previous_tickets: 2,
      avg_resolution_time: 24
    }
  })
});

const { category, priority, suggested_assignee } = await response.json();
// Returns: { category: 'technical', priority: 'high', suggested_assignee: 'support-team-1' }
```

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user_id: 'user123',
    features: {
      failed_login_attempts: 5,
      unusual_activity_score: 0.8,
      access_time_anomaly: true,
      role_escalation_attempts: 2
    }
  })
});

const { risk_level, score, factors } = await response.json();
// Returns: { risk_level: 'high', score: 0.85, factors: ['failed_logins', 'unusual_activity'] }
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user_id: 'user123',
    behavior_data: {
      login_time: '03:00:00',
      login_location: 'Unknown',
      actions_per_minute: 45,
      data_access_pattern: 'unusual'
    }
  })
});

const { is_anomaly, anomaly_score, alert_level } = await response.json();
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user_id: 'user123',
    workload_data: {
      tasks_assigned: 25,
      tasks_completed: 18,
      avg_working_hours: 12,
      overtime_hours: 30,
      missed_deadlines: 5,
      stress_indicators: ['late_submissions', 'quality_issues']
    }
  })
});

const { burnout_risk, score, recommendations } = await response.json();
// Returns suggestions like 'reduce_workload', 'assign_support'
```

### Predictive Project Insights

```javascript
// POST /api/ml/predict-project-delay
const response = await fetch('http://localhost:8000/api/ml/predict-project-delay', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    project_id: 'proj123',
    features: {
      total_tasks: 50,
      completed_tasks: 20,
      remaining_days: 15,
      team_size: 5,
      avg_velocity: 2.5,
      blockers: 3
    }
  })
});

const { delay_probability, estimated_delay_days, risk_factors } = await response.json();
```

## Frontend Integration Patterns

### React Component for User Dashboard

```javascript
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [tickets, setTickets] = useState([]);
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;
  const token = localStorage.getItem('authToken');

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${API_URL}/api/tasks/user/me`, config),
        axios.get(`${API_URL}/api/tickets/user`, config)
      ]);

      // Organize tasks by status
      const organized = {
        todo: tasksRes.data.filter(t => t.status === 'todo'),
        inProgress: tasksRes.data.filter(t => t.status === 'in-progress'),
        done: tasksRes.data.filter(t => t.status === 'done')
      };

      setTasks(organized);
      setTickets(ticketsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching data:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
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
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>

        <div className="column">
          <h2>In Progress ({tasks.inProgress.length})</h2>
          {tasks.inProgress.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>

        <div className="column">
          <h2>Done ({tasks.done.length})</h2>
          {tasks.done.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      </div>

      <div className="tickets-section">
        <h2>My Tickets</h2>
        {tickets.map(ticket => (
          <TicketCard key={ticket._id} ticket={ticket} />
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  const API_URL = process.env.REACT_APP_API_URL;
  const ML_URL = process.env.REACT_APP_ML_API_URL;
  const token = localStorage.getItem('authToken');

  useEffect(() => {
    fetchAnalytics();
    fetchRiskAlerts();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const response = await axios.get(
        `${API_URL}/api/admin/analytics`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setAnalytics(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchRiskAlerts = async () => {
    try {
      const usersRes = await axios.get(
        `${API_URL}/api/users`,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      const riskPromises = usersRes.data.map(user =>
        axios.post(`${ML_URL}/api/ml/predict-risk`, {
          user_id: user._id,
          features: user.securityMetrics
        })
      );

      const risks = await Promise.all(riskPromises);
      const highRisk = risks
        .map((r, i) => ({ ...r.data, user: usersRes.data[i] }))
        .filter(r => r.risk_level === 'high');

      setAlerts(highRisk);
    } catch (error) {
      console.error('Error fetching risks:', error);
    }
  };

  if (!analytics) return <div>Loading...</div>;

  return (
    <div className="admin-analytics">
      <h1>Analytics Dashboard</h1>

      <div className="metrics-grid">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p className="metric-value">{analytics.totalUsers}</p>
        </div>
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p className="metric-value">{analytics.activeTasks}</p>
        </div>
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p className="metric-value">{analytics.openTickets}</p>
        </div>
        <div className="metric-card">
          <h3>At-Risk Users</h3>
          <p className="metric-value alert">{alerts.length}</p>
        </div>
      </div>

      <div className="alerts-section">
        <h2>Security Alerts</h2>
        {alerts.map(alert => (
          <div key={alert.user._id} className="alert-card">
            <h4>{alert.user.name}</h4>
            <p>Risk Score: {(alert.score * 100).toFixed(0)}%</p>
            <p>Factors: {alert.factors.join(', ')}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Common Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Axios Interceptor for Token Management

```javascript
// frontend/src/utils/axios.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.request.use(
  config => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  const API_URL = process.env.REACT_APP_API_URL;
  const token = localStorage.getItem('authToken');

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStart = () => {
    setIsRunning(true);
    axios.post(
      `${API_URL}/api/tasks/${taskId}/time`,
      { action: 'start' },
      { headers: { Authorization: `Bearer ${token}` } }
    );
  };

  const handleStop = async () => {
    setIsRunning(false);
    await axios.post(
      `${API_URL}/api/tasks/${taskId}/time`,
      { action: 'stop', duration: seconds },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setSeconds(0);
  };

  const formatTime = (secs) => {
    const h = Math.floor(secs / 3600);
    const m = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={handleStart}>Start</button>
        ) : (
          <button onClick={handleStop}>Stop</button>
        )}
      </div>
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
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
# or for cloud: mongodb+srv://username:password@cluster.mongodb.net/dbname

# Authentication
JWT_SECRET=your_secure_random_secret_key_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional, for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

### ML Service Configuration

```env
# Model Settings
MODEL_PATH=./models
ENABLE_ONLINE_LEARNING=true

# API Settings
LOG_LEVEL=info
CORS_ORIGINS=http://localhost:3000

# ML Thresholds
RISK_THRESHOLD_HIGH=0.7
RISK_THRESHOLD_MEDIUM=0.4
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.6
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
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

**Common fixes:**
- Ensure MongoDB is running: `mongod --dbpath /path/to/data`
- Check connection string format
- Verify network access if using MongoDB Atlas
- Check firewall settings

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.NODE_ENV === 'production' 
    ? ['https://your-domain.com']
    : ['http://localhost:3000'],
  credentials: true
};

app.use(cors(corsOptions));
```

### ML Model Loading Errors

```python
# ml-service/main.py
import os
from pathlib import Path

MODEL_PATH = Path(os.getenv('MODEL_PATH', './models'))

# Ensure model directory exists
MODEL_PATH.mkdir(parents=True, exist_ok=True)

# Check if model files exist before loading
def load_model_safe(model_name):
    model_file = MODEL_PATH / f"{model_name}.pkl"
    if not model_file.exists():
        print(f"Model {model_name} not found, training new model...")
        return train_new_model(model_name)
    return joblib.load(model_file)
```

### Token Expiration Handling

```javascript
// frontend/src/utils/refreshToken.js
import axios from 'axios';

export const refreshAuthToken = async () => {
  try {
    const refreshToken = localStorage.getItem('refreshToken');
    const response = await axios.post('/api/auth/refresh', {
      refreshToken
    });
    
    localStorage.setItem('authToken', response.data.token);
    return response.data.token;
  } catch (error) {
    localStorage.clear();
    window.location.href = '/login';
  }
};
```

### Performance Optimization for Large User Lists

```javascript
// Use pagination for user lists
const getUsersPaginated = async (page = 1, limit = 20) => {
  const response = await axios.get(
    `${API_URL}/api/users?page=${page}&limit=${limit}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Implement virtual scrolling for large lists
import { FixedSizeList } from 'react-window';

const UserList = ({ users }) => (
  <FixedSizeList
    height={600}
    itemCount={users.length}
    itemSize={80}
    width="100%"
  >
    {({ index, style }) => (
      <div style={style}>
        <UserCard user={users[index]} />
      </div>
    )}
  </FixedSizeList>
);
```

This skill provides comprehensive guidance for working with the Enterprise User Management System with AI Analytics, covering installation, API usage, frontend integration, and common troubleshooting scenarios.
