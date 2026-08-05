---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with task tracking"
  - "implement AI-powered ticket classification system"
  - "build kanban board with time tracking"
  - "add burnout detection and risk prediction"
  - "configure JWT authentication for user management"
  - "integrate ML service for anomaly detection"
  - "create admin dashboard with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform combining task management, support ticketing, and AI-powered analytics for risk detection, burnout analysis, and predictive insights. Built with React, Node.js, FastAPI ML service, and MongoDB.

## What This Project Does

The Enterprise User Management System provides:
- **User & Admin Dashboards**: Role-based interfaces for task management and system administration
- **Task Management**: Kanban board with drag-and-drop, time tracking, and progress monitoring
- **Support Tickets**: AI-powered classification, routing, and priority assignment
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Authentication**: JWT-based secure login with role-based access control
- **Audit Logging**: Track user activities and system events

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB 4.4+

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

Create `.env` file:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
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
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key Backend API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Register
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password123",
  "role": "user"
}

// Verify Token
GET /api/auth/verify
Headers: { "Authorization": "Bearer <token>" }
```

### User Management (Admin)

```javascript
// Get All Users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create User
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update User
PUT /api/users/:userId
{
  "role": "admin",
  "status": "active"
}

// Delete User
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get User Tasks
GET /api/tasks?userId=<userId>&status=<status>

// Create Task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new feature to dashboard",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Update Task Status
PUT /api/tasks/:taskId
{
  "status": "in-progress",
  "timeSpent": 120
}

// Track Time
POST /api/tasks/:taskId/time
{
  "duration": 60,
  "notes": "Completed initial setup"
}
```

### Support Tickets

```javascript
// Create Ticket
POST /api/tickets
{
  "title": "Login Issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get Tickets
GET /api/tickets?status=open&assignedTo=<userId>

// Update Ticket
PUT /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Password reset successful"
}
```

## ML Service API

### AI Ticket Classification

```python
# Python client example
import requests

response = requests.post(
    f"{process.env.ML_SERVICE_URL}/api/ml/classify-ticket",
    json={
        "title": "Cannot login to system",
        "description": "Getting error when trying to authenticate"
    }
)
classification = response.json()
# Returns: {"category": "technical", "priority": "high", "routing": "IT Support"}
```

### Risk Prediction

```python
response = requests.post(
    f"{process.env.ML_SERVICE_URL}/api/ml/predict-risk",
    json={
        "userId": "user123",
        "taskCount": 15,
        "overdueCount": 3,
        "avgCompletionTime": 120,
        "loginFrequency": 0.8
    }
)
risk = response.json()
# Returns: {"riskScore": 0.75, "level": "high", "factors": ["overdue_tasks", "workload"]}
```

### Burnout Detection

```python
response = requests.post(
    f"{process.env.ML_SERVICE_URL}/api/ml/detect-burnout",
    json={
        "userId": "user123",
        "weeklyHours": 55,
        "taskLoad": 20,
        "missedDeadlines": 4,
        "responseTime": 180
    }
)
burnout = response.json()
# Returns: {"burnoutScore": 0.82, "recommendation": "reduce_workload"}
```

### Anomaly Detection

```python
response = requests.post(
    f"{process.env.ML_SERVICE_URL}/api/ml/detect-anomaly",
    json={
        "userId": "user123",
        "loginTime": "03:00",
        "location": "Unknown",
        "failedAttempts": 5,
        "dataAccessPattern": "unusual"
    }
)
anomaly = response.json()
# Returns: {"isAnomaly": true, "confidence": 0.89, "alerts": ["unusual_time", "location"]}
```

## Frontend Integration Examples

### React Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      verifyToken();
    } else {
      setLoading(false);
    }
  }, []);

  const verifyToken = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/verify`);
      setUser(response.data.user);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
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
// components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks?userId=${userId}`
    );
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.put(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
      status: newStatus
    });
    fetchTasks();
  };

  const onDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const onDrop = async (e, status) => {
    const taskId = e.dataTransfer.getData('taskId');
    await moveTask(taskId, status);
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={e => onDrop(e, status)}
          onDragOver={e => e.preventDefault()}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={e => onDragStart(e, task._id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>{task.priority}</span>
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
// components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [risk, burnout, anomalies] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`, {
          userId,
          taskCount: 15,
          overdueCount: 2
        }),
        axios.post(`${process.env.REACT_APP_ML_URL}/api/ml/detect-burnout`, {
          userId,
          weeklyHours: 45
        }),
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/anomalies?userId=${userId}`)
      ]);

      setAnalytics({
        risk: risk.data,
        burnout: burnout.data,
        anomalies: anomalies.data
      });
    } catch (error) {
      console.error('Analytics fetch failed:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.risk.level}`}>
          {(analytics.risk.riskScore * 100).toFixed(0)}%
        </div>
      </div>
      <div className="metric-card">
        <h3>Burnout Detection</h3>
        <div className="score">
          {(analytics.burnout.burnoutScore * 100).toFixed(0)}%
        </div>
        <p>{analytics.burnout.recommendation}</p>
      </div>
      <div className="metric-card">
        <h3>Anomalies</h3>
        <ul>
          {analytics.anomalies.recent.map((a, i) => (
            <li key={i}>{a.type}: {a.description}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.js
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracking Component

```javascript
// components/TimeTracker.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => setSeconds(s => s + 1), 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const saveTime = async () => {
    await axios.post(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
      { duration: seconds }
    );
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="time-tracker">
      <div className="timer">{Math.floor(seconds / 60)}:{seconds % 60}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTime}>Save</button>
    </div>
  );
};

export default TimeTracker;
```

## Backend Database Models

### User Model (MongoDB)

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from river import anomaly, tree
import os

app = FastAPI()

class TicketData(BaseModel):
    title: str
    description: str

class RiskData(BaseModel):
    userId: str
    taskCount: int
    overdueCount: int
    avgCompletionTime: float = 0
    loginFrequency: float = 1.0

class BurnoutData(BaseModel):
    userId: str
    weeklyHours: float
    taskLoad: int
    missedDeadlines: int = 0
    responseTime: float = 60

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, seed=42)

@app.post("/api/ml/classify-ticket")
async def classify_ticket(data: TicketData):
    # Simple keyword-based classification
    text = (data.title + " " + data.description).lower()
    
    category = "general"
    priority = "medium"
    routing = "Support"
    
    if any(word in text for word in ["login", "password", "access", "error"]):
        category = "technical"
        priority = "high"
        routing = "IT Support"
    elif any(word in text for word in ["payment", "invoice", "billing"]):
        category = "billing"
        routing = "Finance"
    elif any(word in text for word in ["feature", "request", "enhancement"]):
        category = "feature_request"
        priority = "low"
        routing = "Product Team"
    
    return {
        "category": category,
        "priority": priority,
        "routing": routing
    }

@app.post("/api/ml/predict-risk")
async def predict_risk(data: RiskData):
    # Calculate risk score based on weighted factors
    overdue_ratio = data.overdueCount / max(data.taskCount, 1)
    
    risk_score = (
        overdue_ratio * 0.4 +
        min(data.taskCount / 20, 1) * 0.3 +
        (1 - data.loginFrequency) * 0.2 +
        min(data.avgCompletionTime / 200, 1) * 0.1
    )
    
    level = "low"
    if risk_score > 0.7:
        level = "high"
    elif risk_score > 0.4:
        level = "medium"
    
    factors = []
    if overdue_ratio > 0.2:
        factors.append("overdue_tasks")
    if data.taskCount > 15:
        factors.append("workload")
    if data.loginFrequency < 0.5:
        factors.append("low_engagement")
    
    return {
        "riskScore": round(risk_score, 2),
        "level": level,
        "factors": factors
    }

@app.post("/api/ml/detect-burnout")
async def detect_burnout(data: BurnoutData):
    # Burnout score calculation
    burnout_score = (
        min(data.weeklyHours / 60, 1) * 0.35 +
        min(data.taskLoad / 25, 1) * 0.25 +
        min(data.missedDeadlines / 5, 1) * 0.25 +
        min(data.responseTime / 300, 1) * 0.15
    )
    
    recommendation = "normal"
    if burnout_score > 0.7:
        recommendation = "reduce_workload"
    elif burnout_score > 0.5:
        recommendation = "monitor_closely"
    
    return {
        "burnoutScore": round(burnout_score, 2),
        "recommendation": recommendation,
        "suggestions": [
            "Distribute tasks more evenly",
            "Schedule time off",
            "Review deadlines"
        ] if burnout_score > 0.5 else []
    }

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(data: dict):
    # Feature engineering for anomaly detection
    features = {
        'login_hour': int(data.get('loginTime', '12:00').split(':')[0]),
        'failed_attempts': data.get('failedAttempts', 0),
        'unusual_location': 1 if data.get('location') == 'Unknown' else 0
    }
    
    # Use online learning anomaly detector
    is_anomaly = (
        features['login_hour'] < 6 or features['login_hour'] > 22 or
        features['failed_attempts'] > 3 or
        features['unusual_location'] == 1
    )
    
    confidence = 0.9 if is_anomaly else 0.1
    
    alerts = []
    if features['login_hour'] < 6 or features['login_hour'] > 22:
        alerts.append("unusual_time")
    if features['unusual_location']:
        alerts.append("location")
    if features['failed_attempts'] > 3:
        alerts.append("failed_login_attempts")
    
    return {
        "isAnomaly": is_anomaly,
        "confidence": confidence,
        "alerts": alerts
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
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

### CORS Configuration

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Timeout Handling

```javascript
// Frontend API client with timeout
const mlClient = axios.create({
  baseURL: process.env.REACT_APP_ML_URL,
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' }
});

mlClient.interceptors.response.use(
  response => response,
  error => {
    if (error.code === 'ECONNABORTED') {
      console.error('ML service timeout');
      return { data: { error: 'Service unavailable' } };
    }
    throw error;
  }
);
```

### JWT Token Refresh

```javascript
// Middleware to refresh expired tokens
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      const refreshToken = localStorage.getItem('refreshToken');
      if (refreshToken) {
        try {
          const response = await axios.post('/api/auth/refresh', { refreshToken });
          localStorage.setItem('token', response.data.token);
          error.config.headers.Authorization = `Bearer ${response.data.token}`;
          return axios(error.config);
        } catch (refreshError) {
          localStorage.clear();
          window.location.href = '/login';
        }
      }
    }
    return Promise.reject(error);
  }
);
```

## Deployment Considerations

### Environment Variables Checklist

Backend:
- `MONGODB_URI`
- `JWT_SECRET`
- `ML_SERVICE_URL`
- `PORT`

ML Service:
- `MODEL_PATH`
- `BACKEND_URL`

Frontend:
- `REACT_APP_API_URL`
- `REACT_APP_ML_URL`

### Production Build

```bash
# Frontend
cd frontend
npm run build

# Backend (PM2)
cd backend
pm2 start npm --name "user-mgmt-backend" -- start

# ML Service (Gunicorn)
cd ml-service
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
```
