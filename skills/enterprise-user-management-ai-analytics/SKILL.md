---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user task management with AI insights"
  - "build admin dashboard with risk detection"
  - "add AI-powered ticket classification"
  - "integrate burnout detection for users"
  - "setup kanban board with time tracking"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI/ML capabilities including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What It Does

This system provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, predictive insights
- **Admin Dashboard**: Organization analytics, audit logs, alerts

## Architecture

The project consists of three main services:
1. **Frontend** (React.js) - User interface on port 3000
2. **Backend** (Node.js) - REST API on port 5000
3. **ML Service** (FastAPI) - AI/ML endpoints on port 8000

## Installation

### Prerequisites

```bash
# Required software
node --version  # v14+ required
python --version  # v3.8+ required
mongod --version  # MongoDB required
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
npm start  # Runs on http://localhost:5000

# ML Service setup (new terminal)
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (new terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

## Configuration

### Backend Environment Variables

Create `backend/.env`:

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management

# Authentication
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
ALLOWED_ORIGINS=http://localhost:3000
```

### ML Service Configuration

Create `ml-service/.env`:

```env
# Server
ML_PORT=8000

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management

# Model paths
RISK_MODEL_PATH=./models/risk_model.pkl
ANOMALY_MODEL_PATH=./models/anomaly_model.pkl
TICKET_MODEL_PATH=./models/ticket_classifier.pkl

# Thresholds
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.75
```

### Frontend Configuration

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Backend API Reference

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
  localStorage.setItem('token', data.token);
  return data;
};

// Get current user
const getCurrentUser = async (token) => {
  const response = await fetch('http://localhost:5000/api/auth/me', {
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
// Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update user
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

// Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// Create task
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
      dueDate: taskData.dueDate,
      priority: taskData.priority,
      status: 'todo'
    })
  });
  return response.json();
};

// Get user tasks
const getUserTasks = async (token) => {
  const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update task status
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

// Track time on task
const trackTime = async (taskId, timeSpent, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return response.json();
};
```

### Support Tickets

```javascript
// Create ticket
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
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// Get user tickets
const getUserTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update ticket status
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API Reference

### Risk Prediction

```python
# Python client example
import requests

def predict_user_risk(user_data, ml_api_url):
    """Predict risk score for a user based on behavior"""
    endpoint = f"{ml_api_url}/api/ml/risk-prediction"
    
    payload = {
        "user_id": user_data["id"],
        "login_frequency": user_data["login_count"],
        "failed_logins": user_data["failed_attempts"],
        "task_completion_rate": user_data["completion_rate"],
        "avg_task_delay": user_data["avg_delay_days"],
        "ticket_count": user_data["support_tickets"]
    }
    
    response = requests.post(endpoint, json=payload)
    return response.json()
    # Returns: {"risk_score": 0.75, "risk_level": "high", "factors": [...]}
```

```javascript
// JavaScript client example
const predictRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: 45,
      failed_logins: 8,
      task_completion_rate: 0.65,
      avg_task_delay: 3.5,
      ticket_count: 12
    })
  });
  return response.json();
};
```

### Anomaly Detection

```python
def detect_anomaly(user_activity, ml_api_url):
    """Detect anomalous behavior patterns"""
    endpoint = f"{ml_api_url}/api/ml/anomaly-detection"
    
    payload = {
        "user_id": user_activity["user_id"],
        "login_time": user_activity["login_timestamp"],
        "ip_address": user_activity["ip"],
        "location": user_activity["location"],
        "device": user_activity["device"],
        "actions_per_minute": user_activity["action_rate"]
    }
    
    response = requests.post(endpoint, json=payload)
    return response.json()
    # Returns: {"is_anomaly": True, "anomaly_score": 0.85, "reason": "..."}
```

### Burnout Detection

```python
def check_burnout(user_id, ml_api_url):
    """Analyze user workload for burnout risk"""
    endpoint = f"{ml_api_url}/api/ml/burnout-detection"
    
    response = requests.post(endpoint, json={
        "user_id": user_id,
        "hours_worked_week": 55,
        "tasks_assigned": 18,
        "tasks_completed": 12,
        "avg_task_complexity": 7.5,
        "overtime_hours": 15,
        "days_since_break": 45
    })
    
    result = response.json()
    return result
    # Returns: {"burnout_risk": 0.82, "level": "high", "recommendations": [...]}
```

### Ticket Classification

```python
def classify_ticket(ticket_text, ml_api_url):
    """Automatically classify and route support ticket"""
    endpoint = f"{ml_api_url}/api/ml/classify-ticket"
    
    response = requests.post(endpoint, json={
        "title": ticket_text["title"],
        "description": ticket_text["description"]
    })
    
    return response.json()
    # Returns: {"category": "technical", "priority": "high", 
    #           "suggested_assignee": "team_id", "confidence": 0.92}
```

### Predictive Analytics

```python
def predict_project_delay(project_data, ml_api_url):
    """Predict likelihood of project delay"""
    endpoint = f"{ml_api_url}/api/ml/project-prediction"
    
    payload = {
        "project_id": project_data["id"],
        "tasks_total": project_data["total_tasks"],
        "tasks_completed": project_data["completed"],
        "team_size": project_data["team_size"],
        "days_remaining": project_data["days_left"],
        "avg_velocity": project_data["velocity"],
        "blockers": project_data["blocker_count"]
    }
    
    response = requests.post(endpoint, json=payload)
    return response.json()
    # Returns: {"delay_probability": 0.68, "estimated_delay_days": 12, 
    #           "suggestions": [...]}
```

## React Frontend Patterns

### Authentication Context

```javascript
// context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchCurrentUser = async () => {
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
    setToken(data.token);
    setUser(data.user);
    localStorage.setItem('token', data.token);
    return data;
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
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
// components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
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

const TaskCard = ({ task, onMove }) => (
  <div className="task-card" draggable>
    <h4>{task.title}</h4>
    <p>{task.description}</p>
    <span className={`priority-${task.priority}`}>{task.priority}</span>
  </div>
);

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalyticsDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AIAnalyticsDashboard = () => {
  const { token, user } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    if (user?.role === 'admin') {
      fetchAnalytics();
    }
  }, [user]);

  const fetchAnalytics = async () => {
    try {
      // Get risk predictions
      const riskResponse = await fetch('http://localhost:8000/api/ml/risk-prediction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: user.id })
      });
      const riskData = await riskResponse.json();

      // Get burnout analysis
      const burnoutResponse = await fetch('http://localhost:8000/api/ml/burnout-detection', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: user.id })
      });
      const burnoutData = await burnoutResponse.json();

      setAnalytics({
        riskScore: riskData,
        burnoutRisk: burnoutData,
        anomalies: []
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <h2>AI Analytics</h2>
      
      {analytics.riskScore && (
        <div className="metric-card">
          <h3>Risk Assessment</h3>
          <div className="risk-score" style={{ 
            color: analytics.riskScore.risk_level === 'high' ? 'red' : 'green' 
          }}>
            {(analytics.riskScore.risk_score * 100).toFixed(1)}%
          </div>
          <p>Level: {analytics.riskScore.risk_level}</p>
        </div>
      )}

      {analytics.burnoutRisk && (
        <div className="metric-card">
          <h3>Burnout Risk</h3>
          <div className="burnout-score">
            {(analytics.burnoutRisk.burnout_risk * 100).toFixed(1)}%
          </div>
          <ul>
            {analytics.burnoutRisk.recommendations?.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Schema Examples

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true
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
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  loginAttempts: { type: Number, default: 0 },
  lastLogin: Date,
  isActive: { type: Boolean, default: true },
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

UserSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
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
    ref: 'User'
  },
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
  timeTracked: { type: Number, default: 0 }, // minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  completedAt: Date
});

module.exports = mongoose.model('Task', TaskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
  title: {
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
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  aiClassified: { type: Boolean, default: false },
  aiConfidence: Number,
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

## ML Model Training Example

```python
# ml-service/train_models.py
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import joblib
import os

def train_risk_model():
    """Train user risk prediction model"""
    # Load historical user data
    data = pd.read_csv('data/user_behavior.csv')
    
    features = [
        'login_frequency',
        'failed_logins',
        'task_completion_rate',
        'avg_task_delay',
        'ticket_count'
    ]
    
    X = data[features]
    y = data['is_risky']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    # Save model
    os.makedirs('models', exist_ok=True)
    joblib.dump(model, 'models/risk_model.pkl')
    
    print(f"Model accuracy: {model.score(X_test, y_test):.2f}")
    return model

def train_ticket_classifier():
    """Train ticket classification model"""
    data = pd.read_csv('data/tickets.csv')
    
    # Use TF-IDF for text features
    from sklearn.feature_extraction.text import TfidfVectorizer
    from sklearn.pipeline import Pipeline
    
    X = data['description']
    y = data['category']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    pipeline = Pipeline([
        ('tfidf', TfidfVectorizer(max_features=1000)),
        ('clf', RandomForestClassifier(n_estimators=100))
    ])
    
    pipeline.fit(X_train, y_train)
    joblib.dump(pipeline, 'models/ticket_classifier.pkl')
    
    print(f"Classifier accuracy: {pipeline.score(X_test, y_test):.2f}")
    return pipeline

if __name__ == '__main__':
    train_risk_model()
    train_ticket_classifier()
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (adminOnly && user.role !== 'admin') return <Navigate to="/dashboard" />;

  return children;
};

export default ProtectedRoute;
```

### API Service Layer

```javascript
// services/api.js
const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

class ApiService {
  constructor() {
    this.token = localStorage.getItem('token');
  }

  setToken(token) {
    this.token = token;
    localStorage.setItem('token', token);
  }

  async request(endpoint, options = {}) {
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers
    };

    if (this.token) {
      headers['Authorization'] = `Bearer ${this.token}`;
    }

    const response = await fetch(`${API_URL}${endpoint}`, {
      ...options,
      headers
    });

    if (!response.ok) {
      throw new Error(`API error: ${response.statusText}`);
    }

    return response.json();
  }

  // User endpoints
  async getUsers() {
    return this.request('/api/users');
  }

  async createUser(userData) {
    return this.request('/api/users', {
      method: 'POST',
      body: JSON.stringify(userData)
    });
  }

  // Task endpoints
  async getTasks() {
    return this.request('/api/tasks/my-tasks');
  }

  async createTask(taskData) {
    return this.request('/api/tasks', {
      method: 'POST',
      body: JSON.stringify(taskData)
    });
  }

  // ML endpoints
  async predictRisk(userId) {
    const response = await fetch(`${ML_API_URL}/api/ml/risk-prediction`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userId })
    });
    return response.json();
  }
}

export default new ApiService();
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection in backend
# backend/config/database.js
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

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

### JWT Authentication Errors

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

module.exports = authMiddleware;
```

### ML Service Not Responding

```bash
# Check if ML service is running
curl http://localhost:8000/health

# Check Python dependencies
cd ml-service
pip list | grep -E 'fastapi|scikit-learn|river'

# Restart with verbose
