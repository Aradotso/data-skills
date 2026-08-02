---
name: enterprise-user-management-ai-analytics
description: Enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "build an enterprise user management system"
  - "implement AI-powered task tracking"
  - "create user dashboard with AI analytics"
  - "set up ticket classification system"
  - "build Kanban board with time tracking"
  - "implement burnout detection for users"
  - "create admin dashboard with AI insights"
  - "add anomaly detection to user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket systems with AI-powered analytics. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights to help organizations improve productivity and automate workflows.

**Key Components:**
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI analytics using scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or remote connection

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
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs at http://localhost:3000
```

## Backend API

### Authentication Endpoints

**Register User**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};
```

**Login**
```javascript
// POST /api/auth/login
const login = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

**Get All Users (Admin)**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management Endpoints

**Create Task**
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      deadline: taskData.deadline
    })
  });
  return response.json();
};
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

**Track Time**
```javascript
// POST /api/tasks/:id/time
const trackTaskTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ 
      timeSpent: timeSpent, // in seconds
      timestamp: new Date().toISOString()
    })
  });
  return response.json();
};
```

### Ticket Management Endpoints

**Create Support Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // optional
    })
  });
  return response.json();
};
```

**Get Tickets**
```javascript
// GET /api/tickets
const getTickets = async (filters = {}) => {
  const token = localStorage.getItem('token');
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

**Update Ticket**
```javascript
// PUT /api/tickets/:id
const updateTicket = async (ticketId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: "Support Request"
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
  return data;
};
```

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginFrequency: userData.loginFrequency,
      failedLogins: userData.failedLogins,
      activityPattern: userData.activityPattern,
      taskCompletionRate: userData.taskCompletionRate
    })
  });
  const data = await response.json();
  // Returns: { riskScore: 0.23, riskLevel: 'low', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivity.userId,
      timestamp: userActivity.timestamp,
      action: userActivity.action,
      ipAddress: userActivity.ipAddress,
      location: userActivity.location,
      deviceInfo: userActivity.deviceInfo
    })
  });
  const data = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.92, reason: 'Unusual location' }
  return data;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      workHours: 50,
      overtimeHours: 15,
      tasksCompleted: 23,
      tasksMissed: 5,
      avgTaskDuration: 4.5,
      stressIndicators: ['late-submissions', 'extended-hours']
    })
  });
  const data = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return data;
};
```

### Project Delay Prediction

```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      teamSize: projectData.teamSize,
      deadline: projectData.deadline,
      currentProgress: projectData.currentProgress,
      blockers: projectData.blockers
    })
  });
  const data = await response.json();
  // Returns: { delayProbability: 0.65, estimatedDelay: '5 days', factors: [...] }
  return data;
};
```

## Frontend Components

### React Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await getUserTasks(userId);
    const grouped = {
      todo: response.filter(t => t.status === 'todo'),
      inProgress: response.filter(t => t.status === 'in-progress'),
      done: response.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus);
    await fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} />
      <Column title="Done" tasks={tasks.done} onMove={moveTask} />
    </div>
  );
};

const Column = ({ title, tasks, onMove }) => (
  <div className="kanban-column">
    <h3>{title}</h3>
    {tasks.map(task => (
      <TaskCard key={task._id} task={task} onMove={onMove} />
    ))}
  </div>
);
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStart = () => setIsRunning(true);
  
  const handleStop = async () => {
    setIsRunning(false);
    await trackTaskTime(taskId, elapsedTime);
    setElapsedTime(0);
  };

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(elapsedTime)}</div>
      <button onClick={isRunning ? handleStop : handleStart}>
        {isRunning ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};
```

### Admin Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    const usersData = await getAllUsers();
    setUsers(usersData);
    
    // Get AI analytics for each user
    const analyticsPromises = usersData.map(async (user) => {
      const risk = await predictUserRisk(user);
      const burnout = await detectBurnout(user._id);
      return { userId: user._id, risk, burnout };
    });
    
    const analyticsData = await Promise.all(analyticsPromises);
    setAnalytics(analyticsData);
  };

  const handleDeleteUser = async (userId) => {
    await deleteUser(userId);
    await loadDashboardData();
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="users-list">
        {users.map(user => {
          const userAnalytics = analytics?.find(a => a.userId === user._id);
          return (
            <div key={user._id} className="user-card">
              <h3>{user.username}</h3>
              <p>{user.email}</p>
              <p>Role: {user.role}</p>
              {userAnalytics && (
                <div className="ai-insights">
                  <span className={`risk-badge ${userAnalytics.risk.riskLevel}`}>
                    Risk: {userAnalytics.risk.riskLevel}
                  </span>
                  <span className={`burnout-badge ${userAnalytics.burnout.burnoutRisk}`}>
                    Burnout: {userAnalytics.burnout.burnoutRisk}
                  </span>
                </div>
              )}
              <button onClick={() => handleDeleteUser(user._id)}>Delete</button>
            </div>
          );
        })}
      </div>
    </div>
  );
};
```

### AI Ticket Assistant Component

```javascript
import React, { useState } from 'react';

const TicketForm = () => {
  const [ticketData, setTicketData] = useState({
    title: '',
    description: ''
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);

  const handleAnalyze = async () => {
    const classification = await classifyTicket(ticketData.description);
    setAiSuggestion(classification);
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    const ticket = {
      ...ticketData,
      category: aiSuggestion?.category,
      priority: aiSuggestion?.priority
    };
    await createTicket(ticket);
    // Reset form
    setTicketData({ title: '', description: '' });
    setAiSuggestion(null);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Title"
        value={ticketData.title}
        onChange={(e) => setTicketData({...ticketData, title: e.target.value})}
      />
      <textarea
        placeholder="Description"
        value={ticketData.description}
        onChange={(e) => setTicketData({...ticketData, description: e.target.value})}
      />
      <button type="button" onClick={handleAnalyze}>
        AI Analyze
      </button>
      
      {aiSuggestion && (
        <div className="ai-suggestion">
          <p>Category: {aiSuggestion.category}</p>
          <p>Priority: {aiSuggestion.priority}</p>
          <p>Confidence: {(aiSuggestion.confidence * 100).toFixed(0)}%</p>
        </div>
      )}
      
      <button type="submit">Submit Ticket</button>
    </form>
  );
};
```

## Common Patterns

### Protected Routes with JWT

```javascript
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  // Decode token to check role
  const payload = JSON.parse(atob(token.split('.')[1]));
  
  if (requireAdmin && payload.role !== 'admin') {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

// Usage in routes
<Route path="/admin" element={
  <ProtectedRoute requireAdmin={true}>
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### Real-time Notifications

```javascript
import { useEffect, useState } from 'react';

const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const fetchNotifications = async () => {
      const token = localStorage.getItem('token');
      const response = await fetch(`http://localhost:5000/api/notifications/${userId}`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setNotifications(data);
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s
    return () => clearInterval(interval);
  }, [userId]);

  return notifications;
};
```

### Batch AI Analytics

```javascript
const performBulkAnalytics = async (userIds) => {
  const results = await Promise.allSettled([
    ...userIds.map(id => predictUserRisk({ userId: id })),
    ...userIds.map(id => detectBurnout(id)),
    ...userIds.map(id => detectAnomaly({ userId: id }))
  ]);

  return {
    risks: results.slice(0, userIds.length),
    burnouts: results.slice(userIds.length, userIds.length * 2),
    anomalies: results.slice(userIds.length * 2)
  };
};
```

## Configuration

### Backend MongoDB Configuration

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

module.exports = authMiddleware;
```

### ML Model Configuration

```python
# ml-service/config.py
import os
from pathlib import Path

class Config:
    MODEL_PATH = Path(os.getenv('MODEL_PATH', './models'))
    LOG_LEVEL = os.getenv('LOG_LEVEL', 'INFO')
    
    # Model thresholds
    RISK_THRESHOLD_HIGH = 0.7
    RISK_THRESHOLD_MEDIUM = 0.4
    
    ANOMALY_THRESHOLD = 0.75
    BURNOUT_THRESHOLD_HIGH = 0.6
    
    # Feature weights
    BURNOUT_WEIGHTS = {
        'overtime_hours': 0.3,
        'tasks_missed': 0.25,
        'work_hours': 0.2,
        'avg_task_duration': 0.15,
        'stress_indicators': 0.1
    }
```

## Troubleshooting

### JWT Token Expired

```javascript
// Implement token refresh
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch('http://localhost:5000/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  
  if (response.ok) {
    const { token } = await response.json();
    localStorage.setItem('token', token);
    return token;
  }
  
  // Redirect to login if refresh fails
  window.location.href = '/login';
};

// Use in API calls
const apiCall = async (url, options) => {
  let response = await fetch(url, options);
  
  if (response.status === 401) {
    await refreshToken();
    // Retry with new token
    options.headers.Authorization = `Bearer ${localStorage.getItem('token')}`;
    response = await fetch(url, options);
  }
  
  return response;
};
```

### ML Service Connection Issues

```javascript
// Add retry logic for ML service calls
const callMLService = async (endpoint, data, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await fetch(`${process.env.REACT_APP_ML_SERVICE_URL}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });
      
      if (response.ok) {
        return await response.json();
      }
    } catch (error) {
      console.error(`ML service call failed (attempt ${i + 1}):`, error);
      if (i === retries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
};
```

### MongoDB Connection Errors

```javascript
// backend/server.js
const connectDB = require('./config/database');

const startServer = async () => {
  try {
    await connectDB();
    
    app.listen(process.env.PORT, () => {
      console.log(`Server running on port ${process.env.PORT}`);
    });
  } catch (error) {
    console.error('Failed to connect to database:', error);
    process.exit(1);
  }
};

startServer();
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Optimization for Large Datasets

```javascript
// Implement pagination for user lists
const getUsersPaginated = async (page = 1, limit = 20) => {
  const token = localStorage.getItem('token');
  const response = await fetch(
    `http://localhost:5000/api/users?page=${page}&limit=${limit}`,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.json();
};

// Implement virtual scrolling for large task lists
import { FixedSizeList } from 'react-window';

const TaskList = ({ tasks }) => (
  <FixedSizeList
    height={600}
    itemCount={tasks.length}
    itemSize={80}
    width="100%"
  >
    {({ index, style }) => (
      <div style={style}>
        <TaskCard task={tasks[index]} />
      </div>
    )}
  </FixedSizeList>
);
```
