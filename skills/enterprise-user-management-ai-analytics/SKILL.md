---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management using React, Node.js, and FastAPI ML service
triggers:
  - set up enterprise user management with AI analytics
  - implement user task management with burnout detection
  - build admin dashboard with AI insights
  - create kanban board with time tracking
  - integrate AI ticket classification system
  - add anomaly detection to user management
  - implement JWT authentication for enterprise app
  - configure ML service for predictive analytics
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights. It provides role-based access control (admin/user), task tracking with Kanban boards, support ticket management, and ML-driven features like risk prediction, anomaly detection, burnout analysis, and project delay prediction.

**Architecture:**
- **Frontend**: React.js application (port 3000)
- **Backend**: Node.js REST API with JWT authentication (port 5000)
- **ML Service**: FastAPI with scikit-learn and River for online learning (port 8000)
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance running

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

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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

Create `ml-service/.env`:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise_users
MODEL_PATH=./models
LOG_LEVEL=info
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

Create `frontend/.env`:
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

**Register User:**
```javascript
const axios = require('axios');

const registerUser = async (userData) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'admin' or 'user'
    });
    return response.data;
  } catch (error) {
    console.error('Registration failed:', error.response.data);
  }
};
```

**Login:**
```javascript
const loginUser = async (email, password) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email,
      password
    });
    
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    return { token, user };
  } catch (error) {
    console.error('Login failed:', error.response.data);
  }
};
```

**Authenticated Request Helper:**
```javascript
const getAuthHeaders = () => {
  const token = localStorage.getItem('token');
  return {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  };
};
```

### User Management (Admin)

**Get All Users:**
```javascript
const getAllUsers = async () => {
  try {
    const response = await axios.get(
      'http://localhost:5000/api/users',
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to fetch users:', error.response.data);
  }
};
```

**Update User:**
```javascript
const updateUser = async (userId, updates) => {
  try {
    const response = await axios.put(
      `http://localhost:5000/api/users/${userId}`,
      updates,
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to update user:', error.response.data);
  }
};
```

**Delete User:**
```javascript
const deleteUser = async (userId) => {
  try {
    const response = await axios.delete(
      `http://localhost:5000/api/users/${userId}`,
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to delete user:', error.response.data);
  }
};
```

### Task Management

**Create Task:**
```javascript
const createTask = async (taskData) => {
  try {
    const response = await axios.post(
      'http://localhost:5000/api/tasks',
      {
        title: taskData.title,
        description: taskData.description,
        assignedTo: taskData.userId,
        priority: taskData.priority, // 'low', 'medium', 'high'
        dueDate: taskData.dueDate,
        status: 'todo' // 'todo', 'in-progress', 'done'
      },
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to create task:', error.response.data);
  }
};
```

**Update Task Status:**
```javascript
const updateTaskStatus = async (taskId, newStatus) => {
  try {
    const response = await axios.patch(
      `http://localhost:5000/api/tasks/${taskId}/status`,
      { status: newStatus },
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to update task:', error.response.data);
  }
};
```

**Get User Tasks:**
```javascript
const getUserTasks = async (userId) => {
  try {
    const response = await axios.get(
      `http://localhost:5000/api/tasks/user/${userId}`,
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to fetch tasks:', error.response.data);
  }
};
```

**Track Time on Task:**
```javascript
const trackTaskTime = async (taskId, timeSpent) => {
  try {
    const response = await axios.post(
      `http://localhost:5000/api/tasks/${taskId}/time`,
      { timeSpent }, // in minutes
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to track time:', error.response.data);
  }
};
```

### Support Tickets

**Create Ticket:**
```javascript
const createTicket = async (ticketData) => {
  try {
    const response = await axios.post(
      'http://localhost:5000/api/tickets',
      {
        title: ticketData.title,
        description: ticketData.description,
        priority: ticketData.priority,
        category: ticketData.category
      },
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to create ticket:', error.response.data);
  }
};
```

**Get Tickets:**
```javascript
const getTickets = async (filters = {}) => {
  try {
    const params = new URLSearchParams(filters);
    const response = await axios.get(
      `http://localhost:5000/api/tickets?${params}`,
      getAuthHeaders()
    );
    return response.data;
  } catch (error) {
    console.error('Failed to fetch tickets:', error.response.data);
  }
};
```

## ML Service API Usage

### AI Ticket Classification

```javascript
const classifyTicket = async (ticketText) => {
  try {
    const response = await axios.post(
      'http://localhost:8000/ai/classify-ticket',
      { text: ticketText }
    );
    
    // Returns: { category: 'technical', priority: 'high', confidence: 0.92 }
    return response.data;
  } catch (error) {
    console.error('Ticket classification failed:', error.response.data);
  }
};
```

### Risk Prediction

```javascript
const predictUserRisk = async (userId) => {
  try {
    const response = await axios.post(
      'http://localhost:8000/ai/risk-prediction',
      { userId }
    );
    
    // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
    return response.data;
  } catch (error) {
    console.error('Risk prediction failed:', error.response.data);
  }
};
```

### Anomaly Detection

```javascript
const detectAnomalies = async (userActivity) => {
  try {
    const response = await axios.post(
      'http://localhost:8000/ai/anomaly-detection',
      {
        userId: userActivity.userId,
        loginTime: userActivity.loginTime,
        location: userActivity.location,
        activityPattern: userActivity.pattern
      }
    );
    
    // Returns: { isAnomaly: true, anomalyScore: 0.88, reasons: [...] }
    return response.data;
  } catch (error) {
    console.error('Anomaly detection failed:', error.response.data);
  }
};
```

### Burnout Detection

```javascript
const detectBurnout = async (userId) => {
  try {
    const response = await axios.post(
      'http://localhost:8000/ai/burnout-detection',
      { userId }
    );
    
    // Returns: { burnoutScore: 0.65, level: 'moderate', recommendations: [...] }
    return response.data;
  } catch (error) {
    console.error('Burnout detection failed:', error.response.data);
  }
};
```

### Predictive Project Insights

```javascript
const getPredictiveInsights = async (projectId) => {
  try {
    const response = await axios.post(
      'http://localhost:8000/ai/project-insights',
      { projectId }
    );
    
    // Returns: { delayProbability: 0.72, estimatedDelay: 5, recommendations: [...] }
    return response.data;
  } catch (error) {
    console.error('Failed to get insights:', error.response.data);
  }
};
```

## React Frontend Patterns

### Authentication Context

```javascript
// src/contexts/AuthContext.js
import React, { createContext, useState, useContext, useEffect } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Verify token and load user
      loadUser();
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    setToken(token);
    setUser(user);
    localStorage.setItem('token', token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  const loadUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      logout();
    }
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`
    );
    
    const grouped = response.data.reduce((acc, task) => {
      const status = task.status === 'in-progress' ? 'inProgress' : task.status;
      if (!acc[status]) acc[status] = [];
      acc[status].push(task);
      return acc;
    }, { todo: [], inProgress: [], done: [] });
    
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus }
    );
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {Object.entries({ todo: 'To Do', inProgress: 'In Progress', done: 'Done' }).map(([key, label]) => (
        <div key={key} className="kanban-column">
          <h3>{label}</h3>
          {tasks[key].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <div className="task-actions">
                {key !== 'done' && (
                  <button onClick={() => moveTask(task._id, 
                    key === 'todo' ? 'in-progress' : 'done')}>
                    Move →
                  </button>
                )}
              </div>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracker Component

```javascript
// src/components/TimeTracker.js
import React, { useState, useEffect, useRef } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
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

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const saveTime = async () => {
    if (seconds > 0) {
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
        { timeSpent: Math.floor(seconds / 60) } // Convert to minutes
      );
      setSeconds(0);
    }
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <div className="controls">
        <button onClick={() => setIsRunning(!isRunning)}>
          {isRunning ? 'Pause' : 'Start'}
        </button>
        <button onClick={() => { setIsRunning(false); setSeconds(0); }}>
          Reset
        </button>
        <button onClick={saveTime} disabled={seconds === 0}>
          Save Time
        </button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

### Admin Dashboard with AI Insights

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [aiInsights, setAiInsights] = useState({});
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    setLoading(true);
    try {
      const usersRes = await axios.get(`${process.env.REACT_APP_API_URL}/api/users`);
      setUsers(usersRes.data);

      // Fetch AI insights for each user
      const insights = {};
      for (const user of usersRes.data) {
        const [riskRes, burnoutRes] = await Promise.all([
          axios.post(`${process.env.REACT_APP_ML_API_URL}/ai/risk-prediction`, { userId: user._id }),
          axios.post(`${process.env.REACT_APP_ML_API_URL}/ai/burnout-detection`, { userId: user._id })
        ]);
        insights[user._id] = {
          risk: riskRes.data,
          burnout: burnoutRes.data
        };
      }
      setAiInsights(insights);
    } catch (error) {
      console.error('Failed to load dashboard:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading dashboard...</div>;

  return (
    <div className="admin-dashboard">
      <h2>Admin Dashboard</h2>
      <div className="users-table">
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Risk Level</th>
              <th>Burnout Score</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td className={`risk-${aiInsights[user._id]?.risk?.riskLevel}`}>
                  {aiInsights[user._id]?.risk?.riskLevel || 'N/A'}
                </td>
                <td>
                  {aiInsights[user._id]?.burnout?.burnoutScore?.toFixed(2) || 'N/A'}
                </td>
                <td>
                  <button onClick={() => viewUserDetails(user._id)}>View</button>
                  <button onClick={() => deleteUser(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );

  async function deleteUser(userId) {
    if (window.confirm('Are you sure?')) {
      await axios.delete(`${process.env.REACT_APP_API_URL}/api/users/${userId}`);
      loadDashboardData();
    }
  }
};

export default AdminDashboard;
```

## Configuration

### Backend Configuration (backend/config.js)

```javascript
module.exports = {
  port: process.env.PORT || 5000,
  mongoUri: process.env.MONGODB_URI || 'mongodb://localhost:27017/enterprise_users',
  jwtSecret: process.env.JWT_SECRET,
  jwtExpire: process.env.JWT_EXPIRE || '7d',
  mlServiceUrl: process.env.ML_SERVICE_URL || 'http://localhost:8000',
  corsOrigin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  
  roles: {
    ADMIN: 'admin',
    USER: 'user'
  },
  
  taskStatuses: ['todo', 'in-progress', 'done'],
  taskPriorities: ['low', 'medium', 'high'],
  
  ticketCategories: ['technical', 'billing', 'general', 'urgent']
};
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    mongodb_uri: str = os.getenv('MONGODB_URI', 'mongodb://localhost:27017/enterprise_users')
    model_path: str = os.getenv('MODEL_PATH', './models')
    log_level: str = os.getenv('LOG_LEVEL', 'info')
    
    # ML Model Parameters
    risk_threshold: float = 0.7
    burnout_threshold: float = 0.6
    anomaly_threshold: float = 0.8
    
    # Feature weights for risk prediction
    task_completion_weight: float = 0.3
    deadline_miss_weight: float = 0.4
    activity_pattern_weight: float = 0.3

settings = Settings()
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user } = useAuth();

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
<Route path="/admin" element={
  <ProtectedRoute requiredRole="admin">
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### Real-time Notifications

```javascript
// src/hooks/useNotifications.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const fetchNotifications = async () => {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/notifications/${userId}`
      );
      setNotifications(response.data);
    };

    fetchNotifications();
    
    // Poll for new notifications every 30 seconds
    const interval = setInterval(fetchNotifications, 30000);
    return () => clearInterval(interval);
  }, [userId]);

  const markAsRead = async (notificationId) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/notifications/${notificationId}/read`
    );
    setNotifications(prev => 
      prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
    );
  };

  return { notifications, markAsRead };
};
```

### Audit Logging Middleware

```javascript
// backend/middleware/auditLog.js
const AuditLog = require('../models/AuditLog');

const auditLog = (action) => async (req, res, next) => {
  res.on('finish', async () => {
    try {
      await AuditLog.create({
        userId: req.user?._id,
        action,
        resource: req.originalUrl,
        method: req.method,
        statusCode: res.statusCode,
        ipAddress: req.ip,
        userAgent: req.get('user-agent'),
        timestamp: new Date()
      });
    } catch (error) {
      console.error('Audit log failed:', error);
    }
  });
  next();
};

// Usage
router.delete('/users/:id', auditLog('DELETE_USER'), deleteUser);
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add axios interceptor to handle token refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Clear token and redirect to login
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues

```javascript
// backend/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected. Attempting to reconnect...');
  connectDB();
});

module.exports = connectDB;
```

### ML Model Not Loading

```python
# ml-service/main.py
import os
import pickle
from fastapi import FastAPI, HTTPException

app = FastAPI()

models = {}

@app.on_event("startup")
async def load_models():
    model_path = os.getenv('MODEL_PATH', './models')
    
    try:
        if os.path.exists(f'{model_path}/risk_model.pkl'):
            with open(f'{model_path}/risk_model.pkl', 'rb') as f:
                models['risk'] = pickle.load(f)
        else:
            print("Risk model not found, using default")
            models['risk'] = None
            
        # Load other models similarly
    except Exception as e:
        print(f"Error loading models: {e}")

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "models_loaded": list(models.keys())
    }
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Performance Optimization for Large Datasets

```javascript
// Implement pagination
const getTasks = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;

  const tasks = await Task.find({ assignedTo: req.user._id })
    .sort({ createdAt: -1 })
    .skip(skip)
    .limit(limit)
    .lean(); // Use lean() for better performance

  const total = await Task.countDocuments({ assignedTo: req.user._id });

  res.json({
    tasks,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
};
```
