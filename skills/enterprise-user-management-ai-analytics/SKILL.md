---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management with AI analytics"
  - "how do I use the AI-powered user management system"
  - "configure user management with anomaly detection"
  - "implement task tracking with burnout analysis"
  - "integrate AI ticket classification system"
  - "deploy enterprise user management with ML service"
  - "create admin dashboard for user analytics"
  - "build user management system with predictive insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform that combines traditional CRUD operations with AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing. Built with React, Node.js, MongoDB, and FastAPI ML service.

## What This Project Does

This system provides a comprehensive enterprise solution for:
- **User & Role Management**: CRUD operations with role-based access control (Admin/User)
- **Task Management**: Kanban-style board with time tracking and status updates
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Security**: JWT authentication with suspicious activity monitoring
- **Audit Logging**: Complete activity tracking for compliance

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Core Architecture

### Backend API (Node.js)

#### User Authentication

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
      role: userData.role || 'user'
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
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

#### User Management (Admin)

```javascript
// GET /api/users - Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
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

#### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
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
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
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

// PUT /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return response.json();
};

// POST /api/tasks/:id/time - Track time
const trackTime = async (taskId, timeEntry, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      startTime: timeEntry.startTime,
      endTime: timeEntry.endTime,
      duration: timeEntry.duration
    })
  });
  return response.json();
};
```

#### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData, token) => {
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
      category: ticketData.category
    })
  });
  const ticket = await response.json();
  
  // AI classification triggers automatically on backend
  return ticket;
};

// GET /api/tickets - Get all tickets (admin)
const getAllTickets = async (token, filters = {}) => {
  const query = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tickets?${query}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### ML Service API (FastAPI)

#### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      userId: userId,
      features: {
        taskCompletionRate: 0.75,
        averageTaskDelay: 2.5,
        ticketCount: 5,
        loginFrequency: 15
      }
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.65, riskLevel: 'medium', factors: [...] }
  return result;
};
```

#### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      userId: activityData.userId,
      loginTime: activityData.loginTime,
      ipAddress: activityData.ipAddress,
      location: activityData.location,
      deviceInfo: activityData.deviceInfo
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, confidence: 0.89, reasons: [...] }
  return result;
};
```

#### Burnout Analysis

```javascript
// POST /api/ml/burnout-analysis
const analyzeBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      userId: userId,
      metrics: {
        tasksAssigned: 25,
        tasksCompleted: 18,
        averageWorkHours: 9.5,
        overtimeHours: 15,
        weekendWork: 4
      }
    })
  });
  const result = await response.json();
  // Returns: { burnoutScore: 0.72, level: 'high', recommendations: [...] }
  return result;
};
```

#### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketContent, token) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketContent.title,
      description: ticketContent.description
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggestedAssignee: 'team-backend' }
  return result;
};
```

#### Project Insights

```javascript
// POST /api/ml/project-insights
const getProjectInsights = async (projectData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasks: projectData.tasks,
      deadline: projectData.deadline,
      teamSize: projectData.teamSize
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.45, estimatedCompletion: '2026-05-15', risks: [...] }
  return result;
};
```

## Frontend Integration Patterns

### React Hook for API Calls

```javascript
// hooks/useAPI.js
import { useState, useEffect } from 'react';

export const useAPI = (url, options = {}) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const token = localStorage.getItem('token');
        const response = await fetch(url, {
          ...options,
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json',
            ...options.headers
          }
        });
        
        if (!response.ok) throw new Error('API request failed');
        
        const result = await response.json();
        setData(result);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [url]);

  return { data, loading, error };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, getNextStatus(column))}>
                Move →
              </button>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

const getNextStatus = (current) => {
  const flow = { todo: 'in-progress', inProgress: 'done', done: 'done' };
  return flow[current];
};

export default KanbanBoard;
```

### Admin Dashboard with AI Insights

```javascript
// components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [aiInsights, setAIInsights] = useState({});
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    // Fetch users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersRes.json();
    setUsers(usersData);

    // Fetch AI insights for each user
    const insights = {};
    for (const user of usersData) {
      const [risk, burnout] = await Promise.all([
        fetch('http://localhost:8000/api/ml/predict-risk', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({ userId: user._id })
        }).then(r => r.json()),
        fetch('http://localhost:8000/api/ml/burnout-analysis', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({ userId: user._id })
        }).then(r => r.json())
      ]);
      
      insights[user._id] = { risk, burnout };
    }
    setAIInsights(insights);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="user-grid">
        {users.map(user => (
          <div key={user._id} className="user-card">
            <h3>{user.name}</h3>
            <p>Email: {user.email}</p>
            <p>Role: {user.role}</p>
            {aiInsights[user._id] && (
              <div className="ai-insights">
                <div className={`risk-badge ${aiInsights[user._id].risk.riskLevel}`}>
                  Risk: {aiInsights[user._id].risk.riskLevel}
                </div>
                <div className={`burnout-badge ${aiInsights[user._id].burnout.level}`}>
                  Burnout: {aiInsights[user._id].burnout.level}
                </div>
              </div>
            )}
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

### ML Service Environment Variables

```bash
# ml-service/.env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
ANOMALY_THRESHOLD=0.7
RISK_THRESHOLD=0.6
BURNOUT_THRESHOLD=0.65
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_JWT_STORAGE_KEY=token
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    return <Navigate to="/login" />;
  }

  // Decode JWT to check role
  const payload = JSON.parse(atob(token.split('.')[1]));
  
  if (requireAdmin && payload.role !== 'admin') {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Real-time Notifications

```javascript
// hooks/useNotifications.js
import { useState, useEffect } from 'react';

export const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    const fetchNotifications = async () => {
      const response = await fetch('http://localhost:5000/api/notifications', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setNotifications(data);
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s

    return () => clearInterval(interval);
  }, []);

  const markAsRead = async (notificationId) => {
    await fetch(`http://localhost:5000/api/notifications/${notificationId}/read`, {
      method: 'PUT',
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setNotifications(prev => prev.filter(n => n._id !== notificationId));
  };

  return { notifications, markAsRead };
};
```

### Time Tracking Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsed, setElapsed] = useState(0);
  const [startTime, setStartTime] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsed(Date.now() - startTime);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning, startTime]);

  const start = () => {
    setStartTime(Date.now());
    setIsRunning(true);
  };

  const stop = async () => {
    setIsRunning(false);
    const endTime = Date.now();
    const duration = Math.floor((endTime - startTime) / 1000);

    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({
        startTime,
        endTime,
        duration
      })
    });

    setElapsed(0);
  };

  const formatTime = (ms) => {
    const seconds = Math.floor(ms / 1000);
    const minutes = Math.floor(seconds / 60);
    const hours = Math.floor(minutes / 60);
    return `${hours.toString().padStart(2, '0')}:${(minutes % 60).toString().padStart(2, '0')}:${(seconds % 60).toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer">{formatTime(elapsed)}</div>
      {!isRunning ? (
        <button onClick={start}>Start</button>
      ) : (
        <button onClick={stop}>Stop</button>
      )}
    </div>
  );
};

export default TimeTracker;
```

## Troubleshooting

### JWT Token Expired

```javascript
// utils/authHandler.js
export const handleAPIError = async (response) => {
  if (response.status === 401) {
    // Token expired, redirect to login
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Session expired');
  }
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message || 'API request failed');
  }
  
  return response.json();
};

// Usage in fetch calls
const data = await fetch(url, options).then(handleAPIError);
```

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
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### ML Model Not Loading

```python
# ml-service/utils/model_loader.py
import os
import pickle
from pathlib import Path

def load_model(model_name):
    model_path = Path(os.getenv('MODEL_PATH', './models')) / f'{model_name}.pkl'
    
    if not model_path.exists():
        # Train and save default model
        from sklearn.ensemble import IsolationForest
        model = IsolationForest(contamination=0.1)
        model_path.parent.mkdir(parents=True, exist_ok=True)
        with open(model_path, 'wb') as f:
            pickle.dump(model, f)
    
    with open(model_path, 'rb') as f:
        return pickle.load(f)
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

### AI Service Unavailable

```javascript
// frontend/utils/aiService.js
export const callAIService = async (endpoint, data, token, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await fetch(`${process.env.REACT_APP_ML_API_URL}${endpoint}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify(data)
      });
      
      if (response.ok) return await response.json();
      
      if (response.status >= 500) {
        // Server error, retry
        await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
        continue;
      }
      
      throw new Error(`AI service error: ${response.status}`);
    } catch (error) {
      if (i === retries - 1) {
        console.error('AI service unavailable, using fallback');
        return { error: 'AI service unavailable', fallback: true };
      }
    }
  }
};
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to effectively assist developers in implementing, configuring, and troubleshooting the system.
