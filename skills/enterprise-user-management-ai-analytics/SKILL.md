---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, anomaly detection, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with user analytics"
  - "add AI-powered risk detection to user system"
  - "build task management with anomaly detection"
  - "integrate ML service for burnout analysis"
  - "deploy user management system with FastAPI ML"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control, user CRUD operations, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment and monitoring
- **Support Tickets**: Smart ticket classification, routing, and tracking
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, performance insights

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation & Setup

### Clone and Install

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MODEL_PATH=./models
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs at http://localhost:3000
```

## Key Backend API Endpoints

### Authentication

```javascript
// Register user (Admin only)
POST /api/auth/register
Body: {
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
Body: {
  "email": "john@example.com",
  "password": "securepass123"
}
Response: {
  "token": "jwt_token_here",
  "user": { "id": "...", "name": "...", "role": "..." }
}

// Get current user
GET /api/auth/me
Headers: { "Authorization": "Bearer <token>" }
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Get user by ID
GET /api/users/:id
Headers: { "Authorization": "Bearer <token>" }

// Update user
PUT /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "name": "Updated Name",
  "email": "updated@example.com",
  "role": "admin"
}

// Delete user (Admin only)
DELETE /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "title": "Implement feature X",
  "description": "Details...",
  "assignedTo": "userId",
  "status": "todo", // todo, in_progress, done
  "priority": "high", // low, medium, high
  "dueDate": "2026-12-31"
}

// Get all tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Update task status
PUT /api/tasks/:id/status
Body: { "status": "in_progress" }

// Track time
POST /api/tasks/:id/time
Body: { "timeSpent": 3600 } // seconds
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "subject": "Login issue",
  "description": "Cannot access my account",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets/my-tickets
Headers: { "Authorization": "Bearer <token>" }

// Update ticket
PUT /api/tickets/:id
Body: {
  "status": "resolved",
  "resolution": "Password reset completed"
}
```

## ML Service API Endpoints

### Risk Prediction

```python
# FastAPI endpoint example
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    lastLoginHoursAgo: float
    taskCompletionRate: float
    avgResponseTimeHours: float

@app.post("/api/ml/predict-risk")
async def predict_risk(data: RiskPredictionRequest):
    # Model inference logic
    features = [
        data.loginAttempts,
        data.failedLogins,
        data.lastLoginHoursAgo,
        data.taskCompletionRate,
        data.avgResponseTimeHours
    ]
    risk_score = model.predict([features])[0]
    
    return {
        "userId": data.userId,
        "riskScore": float(risk_score),
        "riskLevel": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
    }
```

### Anomaly Detection

```python
class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityData: list  # Recent activity metrics

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(data: AnomalyDetectionRequest):
    # Anomaly detection using isolation forest or autoencoder
    is_anomaly = anomaly_detector.predict([data.activityData])[0] == -1
    
    return {
        "userId": data.userId,
        "isAnomaly": is_anomaly,
        "anomalyScore": float(anomaly_detector.score_samples([data.activityData])[0])
    }
```

### Burnout Analysis

```python
class BurnoutAnalysisRequest(BaseModel):
    userId: str
    avgWorkHoursPerDay: float
    tasksOverdue: int
    weekendWorkDays: int
    avgSleepHours: float

@app.post("/api/ml/analyze-burnout")
async def analyze_burnout(data: BurnoutAnalysisRequest):
    features = [
        data.avgWorkHoursPerDay,
        data.tasksOverdue,
        data.weekendWorkDays,
        data.avgSleepHours
    ]
    burnout_probability = burnout_model.predict_proba([features])[0][1]
    
    return {
        "userId": data.userId,
        "burnoutProbability": float(burnout_probability),
        "burnoutLevel": "critical" if burnout_probability > 0.8 else "warning" if burnout_probability > 0.5 else "normal",
        "recommendations": get_burnout_recommendations(burnout_probability)
    }
```

## Frontend Integration Patterns

### Authentication Context

```javascript
// frontend/src/contexts/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchCurrentUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Dashboard Component

```javascript
// frontend/src/components/TaskDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <button onClick={() => updateTaskStatus(task._id, 'in_progress')}>
              Start
            </button>
          </div>
        ))}
      </div>
      <div className="kanban-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <button onClick={() => updateTaskStatus(task._id, 'done')}>
              Complete
            </button>
          </div>
        ))}
      </div>
      <div className="kanban-column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <div key={task._id} className="task-card completed">
            <h4>{task.title}</h4>
          </div>
        ))}
      </div>
    </div>
  );
};

export default TaskDashboard;
```

### AI Analytics Integration

```javascript
// frontend/src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [riskUsers, setRiskUsers] = useState([]);
  const [burnoutUsers, setBurnoutUsers] = useState([]);

  useEffect(() => {
    fetchAIAnalytics();
  }, []);

  const fetchAIAnalytics = async () => {
    try {
      // Fetch risk predictions
      const riskResponse = await axios.get(
        `${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/risk-summary`
      );
      setRiskUsers(riskResponse.data.highRiskUsers);

      // Fetch burnout analysis
      const burnoutResponse = await axios.get(
        `${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/burnout-summary`
      );
      setBurnoutUsers(burnoutResponse.data.atRiskUsers);
    } catch (error) {
      console.error('Failed to fetch AI analytics:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <div className="analytics-section">
        <h3>High Risk Users</h3>
        {riskUsers.map(user => (
          <div key={user.userId} className="alert-card">
            <p>{user.name}</p>
            <span className="risk-badge">{user.riskLevel}</span>
            <p>Score: {user.riskScore.toFixed(2)}</p>
          </div>
        ))}
      </div>
      
      <div className="analytics-section">
        <h3>Burnout Risk</h3>
        {burnoutUsers.map(user => (
          <div key={user.userId} className="alert-card">
            <p>{user.name}</p>
            <span className="burnout-badge">{user.burnoutLevel}</span>
            <p>Probability: {(user.burnoutProbability * 100).toFixed(0)}%</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Backend Model Patterns

### User Model (MongoDB/Mongoose)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'feature_request'],
    default: 'general'
  },
  aiClassification: {
    category: String,
    confidence: Number
  },
  resolution: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Middleware Examples

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    
    if (!req.user || !req.user.isActive) {
      return res.status(401).json({ message: 'User not found or inactive' });
    }
    
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

## ML Model Training Script

```python
# ml-service/train_models.py
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
import joblib
import os

def train_risk_model():
    """Train risk prediction model"""
    # Load historical user data
    data = pd.read_csv('data/user_behavior.csv')
    
    features = ['login_attempts', 'failed_logins', 'last_login_hours_ago', 
                'task_completion_rate', 'avg_response_time_hours']
    X = data[features]
    y = data['is_risky']
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
    
    scaler = StandardScaler()
    X_train_scaled = scaler.fit_transform(X_train)
    X_test_scaled = scaler.transform(X_test)
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train_scaled, y_train)
    
    accuracy = model.score(X_test_scaled, y_test)
    print(f"Risk model accuracy: {accuracy:.2f}")
    
    # Save model and scaler
    os.makedirs('models', exist_ok=True)
    joblib.dump(model, 'models/risk_model.pkl')
    joblib.dump(scaler, 'models/risk_scaler.pkl')

def train_burnout_model():
    """Train burnout detection model"""
    data = pd.read_csv('data/employee_metrics.csv')
    
    features = ['avg_work_hours_per_day', 'tasks_overdue', 'weekend_work_days', 'avg_sleep_hours']
    X = data[features]
    y = data['burnout']
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
    
    scaler = StandardScaler()
    X_train_scaled = scaler.fit_transform(X_train)
    X_test_scaled = scaler.transform(X_test)
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train_scaled, y_train)
    
    accuracy = model.score(X_test_scaled, y_test)
    print(f"Burnout model accuracy: {accuracy:.2f}")
    
    joblib.dump(model, 'models/burnout_model.pkl')
    joblib.dump(scaler, 'models/burnout_scaler.pkl')

def train_anomaly_detector():
    """Train anomaly detection model"""
    data = pd.read_csv('data/normal_activity.csv')
    
    features = ['actions_per_hour', 'data_access_count', 'login_frequency', 
                'unusual_hours_activity']
    X = data[features]
    
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)
    
    model = IsolationForest(contamination=0.1, random_state=42)
    model.fit(X_scaled)
    
    joblib.dump(model, 'models/anomaly_detector.pkl')
    joblib.dump(scaler, 'models/anomaly_scaler.pkl')
    print("Anomaly detector trained")

if __name__ == '__main__':
    train_risk_model()
    train_burnout_model()
    train_anomaly_detector()
```

## Configuration Best Practices

### Backend Configuration

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

### ML Service Configuration

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    mongodb_uri: str = os.getenv('MONGODB_URI', 'mongodb://localhost:27017/enterprise_user_management')
    model_path: str = os.getenv('MODEL_PATH', './models')
    risk_threshold: float = 0.7
    burnout_threshold: float = 0.8
    
    class Config:
        env_file = '.env'

settings = Settings()
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add token refresh logic
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Service Connection Issues

```python
# Add health check endpoint
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "models_loaded": {
            "risk": os.path.exists(f"{settings.model_path}/risk_model.pkl"),
            "burnout": os.path.exists(f"{settings.model_path}/burnout_model.pkl"),
            "anomaly": os.path.exists(f"{settings.model_path}/anomaly_detector.pkl")
        }
    }
```

### MongoDB Connection Errors

```javascript
// Add retry logic
const connectWithRetry = async (retries = 5) => {
  for (let i = 0; i < retries; i++) {
    try {
      await mongoose.connect(process.env.MONGODB_URI);
      console.log('MongoDB connected');
      return;
    } catch (error) {
      console.log(`MongoDB connection attempt ${i + 1} failed`);
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
  throw new Error('Could not connect to MongoDB');
};
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

This skill provides comprehensive guidance for working with the Enterprise User Management System with AI Analytics across frontend, backend, and ML components.
