---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management with AI analytics"
  - "how do I use the AI-powered user management system"
  - "configure enterprise user management with ML features"
  - "implement AI analytics for user management"
  - "integrate risk detection and anomaly detection in user system"
  - "build user management dashboard with AI insights"
  - "add AI-based ticket classification to my app"
  - "create enterprise task management with predictive analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system that combines user administration, task tracking, support ticket management, and AI-powered analytics. It provides risk detection, anomaly detection, burnout analysis, predictive project insights, and AI-based ticket routing.

## What It Does

- **User Management**: Role-based access control, authentication via JWT, user CRUD operations
- **Task Management**: Kanban boards (To Do → In Progress → Done), time tracking, task assignment
- **Support Tickets**: Raise, track, and AI-classify support requests
- **AI Analytics**: 
  - Risk prediction based on user behavior
  - Anomaly detection for security
  - Burnout detection using workload analysis
  - Predictive project insights (delay prediction)
  - AI-based ticket classification and routing
  - AI assistant for system queries

## Architecture

The system consists of three main components:
1. **Frontend** (React.js) - User/Admin dashboards
2. **Backend** (Node.js/Express) - REST APIs, authentication, business logic
3. **ML Service** (FastAPI + scikit-learn + River) - AI/ML endpoints

## Installation

### Prerequisites

- Node.js (v14+)
- Python 3.8+
- MongoDB

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

Create `.env` file in `backend/`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/`:

```env
MODEL_PATH=./models
DB_CONNECTION_STRING=mongodb://localhost:27017/enterprise_ums
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Frontend runs at `http://localhost:3000`

## Backend API Reference

### Authentication

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
      role: userData.role // 'user' or 'admin'
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
  // Store token for subsequent requests
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

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

### Task Management

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
      status: 'todo', // 'todo', 'inprogress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
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
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status }) // 'todo', 'inprogress', 'done'
  });
  return response.json();
};
```

### Support Tickets

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
      category: ticketData.category, // Optional
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API Reference

### Risk Prediction

```javascript
// POST /api/ml/risk-prediction
const predictRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        failed_login_attempts: 3,
        unusual_access_time: true,
        location_change: false,
        data_access_volume: 150
      }
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomaly = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: behaviorData.userId,
      metrics: {
        login_time: behaviorData.loginTime,
        session_duration: behaviorData.sessionDuration,
        actions_per_minute: behaviorData.actionsPerMinute,
        ip_address: behaviorData.ipAddress
      }
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, details: {...} }
  return data;
};
```

### Burnout Detection

```javascript
// POST /api/ml/burnout-detection
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      workload_metrics: {
        tasks_assigned: 25,
        tasks_completed: 18,
        avg_task_duration: 4.5,
        overtime_hours: 15,
        weekend_work: true
      }
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.82, recommendations: [...] }
  return data;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketText.title,
      description: ticketText.description
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', assigned_team: 'IT' }
  return data;
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/project-prediction
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      metrics: {
        total_tasks: 50,
        completed_tasks: 20,
        days_elapsed: 30,
        total_days: 90,
        team_size: 5,
        blockers: 3
      }
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.65, expected_completion: '2026-07-15', recommendations: [...] }
  return data;
};
```

## React Frontend Integration

### Authentication Context

```javascript
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and fetch user data
      fetchUserProfile();
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/profile`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      logout();
    }
  };

  const login = async (credentials) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    const data = await response.json();
    setToken(data.token);
    setUser(data.user);
    localStorage.setItem('token', data.token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { user, token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${user.id}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inprogress: data.filter(t => t.status === 'inprogress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      method: 'PATCH',
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
      {['todo', 'inprogress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => {
                const nextStatus = status === 'todo' ? 'inprogress' : 'done';
                if (status !== 'done') updateTaskStatus(task.id, nextStatus);
              }}>
                {status === 'todo' ? 'Start' : status === 'inprogress' ? 'Complete' : '✓'}
              </button>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AIAnalyticsDashboard = () => {
  const { user, token } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    anomalies: [],
    burnoutRisk: null
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    // Fetch risk prediction
    const riskResponse = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: user.id, features: {} })
    });
    const riskData = await riskResponse.json();

    // Fetch burnout detection
    const burnoutResponse = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: user.id, workload_metrics: {} })
    });
    const burnoutData = await burnoutResponse.json();

    setAnalytics({
      riskScore: riskData.risk_score,
      burnoutRisk: burnoutData.burnout_risk,
      anomalies: []
    });
  };

  return (
    <div className="analytics-dashboard">
      <h2>AI Analytics</h2>
      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>Risk Score</h3>
          <p className={`score ${analytics.riskScore > 0.7 ? 'high' : 'low'}`}>
            {analytics.riskScore ? (analytics.riskScore * 100).toFixed(0) : 'N/A'}%
          </p>
        </div>
        <div className="analytics-card">
          <h3>Burnout Risk</h3>
          <p className={`risk-level ${analytics.burnoutRisk}`}>
            {analytics.burnoutRisk || 'N/A'}
          </p>
        </div>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Common Patterns

### Protected Routes

```javascript
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### API Error Handling

```javascript
const apiCall = async (url, options = {}) => {
  try {
    const response = await fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      }
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'API request failed');
    }

    return await response.json();
  } catch (error) {
    console.error('API Error:', error);
    throw error;
  }
};
```

### Real-time Notifications

```javascript
import React, { useEffect, useState, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const NotificationListener = () => {
  const { user, token } = useContext(AuthContext);
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    // Poll for new notifications every 30 seconds
    const interval = setInterval(async () => {
      const response = await fetch(
        `${process.env.REACT_APP_API_URL}/api/notifications/user/${user.id}`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      const data = await response.json();
      setNotifications(data.notifications);
    }, 30000);

    return () => clearInterval(interval);
  }, [user, token]);

  return (
    <div className="notifications">
      {notifications.map(notif => (
        <div key={notif.id} className="notification">
          {notif.message}
        </div>
      ))}
    </div>
  );
};

export default NotificationListener;
```

## Configuration

### Backend Configuration (backend/config.js)

```javascript
module.exports = {
  port: process.env.PORT || 5000,
  mongodb: {
    uri: process.env.MONGODB_URI || 'mongodb://localhost:27017/enterprise_ums',
    options: {
      useNewUrlParser: true,
      useUnifiedTopology: true
    }
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: '24h'
  },
  mlService: {
    url: process.env.ML_SERVICE_URL || 'http://localhost:8000'
  }
};
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    MODEL_PATH = os.getenv('MODEL_PATH', './models')
    DB_CONNECTION_STRING = os.getenv('DB_CONNECTION_STRING', 'mongodb://localhost:27017/enterprise_ums')
    RISK_THRESHOLD = float(os.getenv('RISK_THRESHOLD', '0.7'))
    ANOMALY_THRESHOLD = float(os.getenv('ANOMALY_THRESHOLD', '0.8'))
    BURNOUT_THRESHOLD = float(os.getenv('BURNOUT_THRESHOLD', '0.75'))
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Add token refresh logic
const refreshToken = async () => {
  const oldToken = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/refresh`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${oldToken}`
    }
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

### CORS Issues

Backend CORS configuration:

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### MongoDB Connection Errors

```javascript
// backend/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Model Loading Issues

```python
# ml-service/models/loader.py
import os
import pickle
import logging

logger = logging.getLogger(__name__)

def load_model(model_name):
    model_path = os.path.join(os.getenv('MODEL_PATH', './models'), f'{model_name}.pkl')
    try:
        with open(model_path, 'rb') as f:
            model = pickle.load(f)
        logger.info(f"Model {model_name} loaded successfully")
        return model
    except FileNotFoundError:
        logger.error(f"Model {model_name} not found at {model_path}")
        # Return a default model or retrain
        return None
```

### API Rate Limiting

```javascript
// backend/middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');

const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP, please try again later.'
});

module.exports = apiLimiter;
```

## Performance Optimization

### Caching ML Predictions

```python
# ml-service/cache.py
from functools import lru_cache
import hashlib
import json

@lru_cache(maxsize=1000)
def cached_prediction(model_name, features_hash):
    # Cache predictions for identical feature sets
    pass

def get_prediction_with_cache(model, features):
    features_hash = hashlib.md5(json.dumps(features, sort_keys=True).encode()).hexdigest()
    return cached_prediction(model.__class__.__name__, features_hash)
```

### Database Indexing

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true, index: true },
  email: { type: String, required: true, unique: true, index: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user', index: true },
  createdAt: { type: Date, default: Date.now }
});

// Compound index for queries
userSchema.index({ role: 1, createdAt: -1 });

module.exports = mongoose.model('User', userSchema);
```
