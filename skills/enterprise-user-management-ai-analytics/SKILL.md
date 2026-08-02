---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered task management"
  - "create user dashboard with kanban board"
  - "add AI ticket classification and routing"
  - "build admin analytics dashboard"
  - "integrate burnout detection and risk prediction"
  - "deploy user management with ML service"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js, FastAPI, and MongoDB.

## What It Does

This system provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Ticketing**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring

## Installation

### Prerequisites

- Node.js 14+
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

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=info
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs on http://localhost:3000
```

## Key Components

### Backend API Endpoints

#### Authentication

```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response
{
  "token": "jwt_token_here",
  "user": {
    "id": "user_id",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

#### User Management (Admin Only)

```javascript
// GET /api/users - Get all users
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user

// Example: Update user
fetch('http://localhost:5000/api/users/123', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    name: "Jane Doe",
    role: "admin"
  })
});
```

#### Task Management

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement login feature",
  "description": "Add JWT authentication",
  "assignedTo": "user_id",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id - Update task status
{
  "status": "in-progress"
}

// GET /api/tasks/user/:userId - Get user tasks
// DELETE /api/tasks/:id - Delete task
```

#### Support Tickets

```javascript
// POST /api/tickets - Create ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when logging in",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - Get all tickets (admin)
// GET /api/tickets/user/:userId - Get user tickets
// PUT /api/tickets/:id - Update ticket
{
  "status": "resolved",
  "resolution": "Reset user permissions"
}
```

### ML Service API Endpoints

#### Ticket Classification

```python
# POST /api/ml/classify-ticket
import requests

response = requests.post('http://localhost:8000/api/ml/classify-ticket', 
  json={
    "title": "Cannot login to system",
    "description": "Getting authentication error",
    "userId": "user123"
  },
  headers={'Content-Type': 'application/json'}
)

# Response
{
  "category": "technical",
  "priority": "high",
  "suggestedAssignee": "support_team_id",
  "confidence": 0.87
}
```

#### Risk Prediction

```python
# POST /api/ml/predict-risk
response = requests.post('http://localhost:8000/api/ml/predict-risk',
  json={
    "userId": "user123",
    "recentActivity": [
      {"action": "login_failure", "timestamp": "2026-04-15T10:00:00Z"},
      {"action": "access_denied", "timestamp": "2026-04-15T10:05:00Z"}
    ],
    "taskMetrics": {
      "overdueTasks": 5,
      "completionRate": 0.45
    }
  }
)

# Response
{
  "riskLevel": "high",
  "riskScore": 0.78,
  "factors": ["multiple_login_failures", "low_task_completion"],
  "recommendations": ["review_access_permissions", "workload_assessment"]
}
```

#### Burnout Detection

```python
# POST /api/ml/detect-burnout
response = requests.post('http://localhost:8000/api/ml/detect-burnout',
  json={
    "userId": "user123",
    "workload": {
      "tasksAssigned": 25,
      "avgHoursPerDay": 11.5,
      "overtimeHours": 20,
      "missedDeadlines": 8
    },
    "performance": {
      "completionRate": 0.55,
      "qualityScore": 0.60
    }
  }
)

# Response
{
  "burnoutRisk": "high",
  "burnoutScore": 0.82,
  "indicators": ["excessive_hours", "declining_quality", "missed_deadlines"],
  "suggestions": ["reduce_workload", "schedule_break", "redistribute_tasks"]
}
```

#### Project Delay Prediction

```python
# POST /api/ml/predict-delay
response = requests.post('http://localhost:8000/api/ml/predict-delay',
  json={
    "projectId": "proj123",
    "teamSize": 5,
    "tasksCompleted": 15,
    "tasksRemaining": 30,
    "daysRemaining": 10,
    "avgCompletionRate": 2.1,
    "complexity": "high"
  }
)

# Response
{
  "delayProbability": 0.75,
  "estimatedDelay": 7,
  "completionDate": "2026-05-08",
  "factors": ["high_complexity", "low_completion_rate"],
  "mitigationSteps": ["add_resources", "reduce_scope", "extend_deadline"]
}
```

## Frontend Usage Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Verify token and load user
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);

  const login = async (email, password) => {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/auth/login`,
      { email, password }
    );
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`
    );
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.put(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}`,
      { status: newStatus }
    );
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.toUpperCase()}</h3>
          {tasks[column].map(task => (
            <TaskCard
              key={task.id}
              task={task}
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      ))}
    </div>
  );
};
```

### AI Analytics Dashboard

```javascript
// src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    checkAnomalies();
  }, []);

  const fetchAnalytics = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/analytics/dashboard`
    );
    setAnalytics(response.data);
  };

  const checkAnomalies = async () => {
    const users = await axios.get(`${process.env.REACT_APP_API_URL}/users`);
    const anomalyChecks = users.data.map(async (user) => {
      const risk = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`,
        {
          userId: user.id,
          recentActivity: user.recentActivity,
          taskMetrics: user.taskMetrics
        }
      );
      if (risk.data.riskLevel === 'high') {
        return { user, risk: risk.data };
      }
    });
    const results = await Promise.all(anomalyChecks);
    setAlerts(results.filter(Boolean));
  };

  return (
    <div className="analytics-dashboard">
      <h2>Organization Analytics</h2>
      {alerts.length > 0 && (
        <div className="alerts">
          <h3>High Risk Users</h3>
          {alerts.map(alert => (
            <AlertCard key={alert.user.id} alert={alert} />
          ))}
        </div>
      )}
    </div>
  );
};
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStart = () => setIsRunning(true);
  const handleStop = async () => {
    setIsRunning(false);
    await axios.post(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
      { seconds }
    );
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={isRunning ? handleStop : handleStart}>
        {isRunning ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { 
    type: String, 
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
});
```

### ML Model Training Script

```python
# ml-service/train_models.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import joblib
import pandas as pd

def train_ticket_classifier():
    # Load historical ticket data
    data = pd.read_csv('data/tickets.csv')
    
    X = data[['title_length', 'description_length', 'hour_created']]
    y = data['category']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    print(f'Model accuracy: {accuracy}')
    
    joblib.dump(model, 'models/ticket_classifier.pkl')

def train_risk_predictor():
    data = pd.read_csv('data/user_activity.csv')
    
    features = ['login_failures', 'access_denials', 'overdue_tasks', 
                'completion_rate', 'avg_response_time']
    X = data[features]
    y = data['is_risky']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    model = RandomForestClassifier(n_estimators=150, random_state=42)
    model.fit(X_train, y_train)
    
    joblib.dump(model, 'models/risk_predictor.pkl')

if __name__ == '__main__':
    train_ticket_classifier()
    train_risk_predictor()
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

// Usage in App.js
<Route 
  path="/admin" 
  element={
    <ProtectedRoute requireAdmin={true}>
      <AdminDashboard />
    </ProtectedRoute>
  } 
/>
```

### Axios Interceptor Setup

```javascript
// src/utils/axiosConfig.js
import axios from 'axios';

axios.defaults.baseURL = process.env.REACT_APP_API_URL;

axios.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### Real-time Notifications

```javascript
// src/hooks/useNotifications.js
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const socket = io(process.env.REACT_APP_API_URL);
    
    socket.emit('subscribe', userId);
    
    socket.on('notification', (notification) => {
      setNotifications(prev => [notification, ...prev]);
    });

    return () => socket.disconnect();
  }, [userId]);

  return notifications;
};
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Backend: Refresh token endpoint
router.post('/refresh-token', async (req, res) => {
  const { refreshToken } = req.body;
  
  try {
    const decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    const user = await User.findById(decoded.id);
    
    const newToken = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token: newToken });
  } catch (error) {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});
```

### ML Service Connection Issues

```python
# ml-service/main.py - Add health check
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### MongoDB Connection Retry

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async (retries = 5) => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected');
  } catch (error) {
    if (retries > 0) {
      console.log(`Retrying connection... (${retries} attempts left)`);
      setTimeout(() => connectDB(retries - 1), 5000);
    } else {
      console.error('MongoDB connection failed:', error);
      process.exit(1);
    }
  }
};

module.exports = connectDB;
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

### Environment Variables Not Loading

```javascript
// backend/server.js - Add at top
require('dotenv').config();

if (!process.env.JWT_SECRET) {
  console.error('FATAL ERROR: JWT_SECRET is not defined');
  process.exit(1);
}
```

## Testing

### API Testing Example

```javascript
// backend/tests/auth.test.js
const request = require('supertest');
const app = require('../server');

describe('Authentication', () => {
  it('should register a new user', async () => {
    const res = await request(app)
      .post('/api/auth/register')
      .send({
        name: 'Test User',
        email: 'test@example.com',
        password: 'password123'
      });
    
    expect(res.statusCode).toBe(201);
    expect(res.body).toHaveProperty('token');
  });

  it('should login existing user', async () => {
    const res = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password123'
      });
    
    expect(res.statusCode).toBe(200);
    expect(res.body).toHaveProperty('token');
  });
});
```

This enterprise user management system provides a complete solution for managing users, tasks, and support with AI-powered insights to improve productivity and decision-making.
