---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user task management with AI insights"
  - "build admin dashboard with risk detection"
  - "add ticket classification and routing system"
  - "integrate burnout detection for users"
  - "configure JWT authentication for user management"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user administration, task tracking, and support ticket management with machine learning capabilities. The system provides AI-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Architecture:**
- **Frontend:** React.js application for admin and user dashboards
- **Backend:** Node.js REST API with JWT authentication
- **ML Service:** FastAPI service with scikit-learn and River for online learning
- **Database:** MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Required tools
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend server
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management Endpoints

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'in-progress', 'done'
  });
  return response.json();
};
```

### Ticket Management Endpoints

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (Admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### AI-Powered Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: "Issue with login"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.89 }
  return result;
};
```

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userMetrics) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userMetrics.userId,
      loginFrequency: userMetrics.loginFrequency,
      taskCompletionRate: userMetrics.taskCompletionRate,
      averageResponseTime: userMetrics.avgResponseTime,
      failedLoginAttempts: userMetrics.failedAttempts
    })
  });
  const result = await response.json();
  // Returns: { riskLevel: 'high', score: 0.78, factors: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      tasksAssigned: workloadData.totalTasks,
      tasksCompleted: workloadData.completedTasks,
      averageWorkHours: workloadData.avgHours,
      overtimeHours: workloadData.overtime,
      weekendWork: workloadData.weekendWork
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'moderate', score: 0.65, recommendations: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityLog) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityLog.userId,
      timestamp: activityLog.timestamp,
      action: activityLog.action,
      ipAddress: activityLog.ip,
      location: activityLog.location
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.92, reason: 'Unusual login location' }
  return result;
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/predict-project-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-project-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      daysRemaining: projectData.daysLeft,
      teamSize: projectData.teamSize,
      complexity: projectData.complexity
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.73, estimatedDelay: 5, recommendations: [...] }
  return result;
};
```

## React Frontend Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      // Verify token and fetch user data
      fetchUserProfile();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
      return { success: true };
    }
    return { success: false, error: data.message };
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId, token }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    
    fetchTasks();
  };

  const renderColumn = (title, status, taskList) => (
    <div 
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => handleDrop(e, status)}
    >
      <h3>{title}</h3>
      {taskList.map(task => (
        <div
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => handleDragStart(e, task._id)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('To Do', 'todo', tasks.todo)}
      {renderColumn('In Progress', 'in-progress', tasks.inProgress)}
      {renderColumn('Done', 'done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard with AI Insights

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';

const AdminDashboard = ({ token }) => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchAlerts();
  }, []);

  const fetchAnalytics = async () => {
    const response = await fetch('http://localhost:5000/api/admin/analytics', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };

  const fetchAlerts = async () => {
    const response = await fetch('http://localhost:8000/api/ml/alerts', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAlerts(data.alerts);
  };

  const analyzeUserRisk = async (userId) => {
    const userMetrics = await fetch(
      `http://localhost:5000/api/users/${userId}/metrics`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    ).then(r => r.json());

    const riskAnalysis = await fetch('http://localhost:8000/api/ml/predict-risk', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(userMetrics)
    }).then(r => r.json());

    return riskAnalysis;
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{analytics?.totalUsers || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{analytics?.activeTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{analytics?.openTickets || 0}</p>
        </div>
      </div>

      <div className="alerts-section">
        <h2>AI-Powered Alerts</h2>
        {alerts.map(alert => (
          <div key={alert.id} className={`alert alert-${alert.severity}`}>
            <strong>{alert.type}:</strong> {alert.message}
            <span className="timestamp">{alert.timestamp}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Backend Configuration (backend/config.js)

```javascript
module.exports = {
  port: process.env.PORT || 5000,
  mongoUri: process.env.MONGODB_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpire: process.env.JWT_EXPIRE || '7d',
  mlServiceUrl: process.env.ML_SERVICE_URL || 'http://localhost:8000',
  corsOrigin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  maxLoginAttempts: 5,
  lockoutDuration: 15 * 60 * 1000, // 15 minutes
  fileUploadLimit: '10mb'
};
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from typing import List

class Settings:
    MODEL_PATH: str = os.getenv("MODEL_PATH", "./models")
    LOG_LEVEL: str = os.getenv("LOG_LEVEL", "info")
    CORS_ORIGINS: List[str] = os.getenv(
        "CORS_ORIGINS", 
        "http://localhost:3000,http://localhost:5000"
    ).split(",")
    
    # ML Model thresholds
    RISK_THRESHOLD_HIGH: float = 0.7
    RISK_THRESHOLD_MEDIUM: float = 0.4
    BURNOUT_THRESHOLD: float = 0.6
    ANOMALY_THRESHOLD: float = 0.8

settings = Settings()
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracking Hook

```javascript
// src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';

const useTimeTracker = (taskId, token) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }
    return () => clearInterval(intervalRef.current);
  }, [isRunning]);

  const start = () => setIsRunning(true);
  
  const pause = () => setIsRunning(false);
  
  const stop = async () => {
    setIsRunning(false);
    // Save time to backend
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ timeSpent: seconds })
    });
    setSeconds(0);
  };

  const formatTime = () => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return { seconds, isRunning, start, pause, stop, formatTime };
};

export default useTimeTracker;
```

### Notification System

```javascript
// src/utils/notifications.js
export const sendNotification = async (userId, message, type, token) => {
  await fetch('http://localhost:5000/api/notifications', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId,
      message,
      type, // 'info', 'warning', 'error', 'success'
      timestamp: new Date().toISOString()
    })
  });
};

export const subscribeToNotifications = (userId, token, callback) => {
  // WebSocket or polling implementation
  const eventSource = new EventSource(
    `http://localhost:5000/api/notifications/stream?userId=${userId}&token=${token}`
  );
  
  eventSource.onmessage = (event) => {
    const notification = JSON.parse(event.data);
    callback(notification);
  };
  
  return () => eventSource.close();
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongodb

# Start MongoDB
sudo systemctl start mongodb

# Check connection string
echo $MONGODB_URI
# Should be: mongodb://localhost:27017/enterprise-ums
```

### JWT Token Expiration

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const response = await fetch('http://localhost:5000/api/auth/refresh', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};
```

### ML Service Not Responding

```bash
# Check if Python dependencies are installed
pip list | grep -E 'fastapi|scikit-learn|river'

# Reinstall if needed
pip install -r ml-service/requirements.txt

# Check if service is running
curl http://localhost:8000/health

# View logs
tail -f ml-service/logs/app.log
```

### CORS Issues

```javascript
// Backend: Update CORS configuration (backend/server.js)
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH']
}));
```

### Performance Optimization

```javascript
// Implement pagination for large datasets
const getPaginatedUsers = async (page = 1, limit = 20, token) => {
  const response = await fetch(
    `http://localhost:5000/api/users?page=${page}&limit=${limit}`,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.json();
};

// Implement caching for frequent requests
const cache = new Map();

const getCachedData = async (key, fetchFn, ttl = 60000) => {
  const cached = cache.get(key);
  if (cached && Date.now() - cached.timestamp < ttl) {
    return cached.data;
  }
  
  const data = await fetchFn();
  cache.set(key, { data, timestamp: Date.now() });
  return data;
};
```

### Error Handling Pattern

```javascript
// Centralized error handler
const handleApiError = (error) => {
  if (error.response) {
    switch (error.response.status) {
      case 401:
        // Unauthorized - redirect to login
        window.location.href = '/login';
        break;
      case 403:
        // Forbidden
        alert('You do not have permission to perform this action');
        break;
      case 404:
        // Not found
        alert('Resource not found');
        break;
      case 500:
        // Server error
        alert('Server error. Please try again later');
        break;
      default:
        alert(error.response.data.message || 'An error occurred');
    }
  } else {
    alert('Network error. Please check your connection');
  }
};

// Usage in components
try {
  const data = await fetchUserData();
} catch (error) {
  handleApiError(error);
}
```

## Deployment

### Production Environment Variables

```bash
# Backend .env.production
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=${ML_SERVICE_URL}
NODE_ENV=production
CORS_ORIGIN=${FRONTEND_URL}

# Frontend .env.production
REACT_APP_API_URL=${API_URL}
REACT_APP_ML_URL=${ML_SERVICE_URL}
```

### Build for Production

```bash
# Build frontend
cd frontend
npm run build

# Build backend (if needed)
cd backend
npm run build

# Start production server
NODE_ENV=production npm start
```
