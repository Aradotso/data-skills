---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and workforce insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create task management with burnout detection"
  - "build ticket classification system"
  - "add risk prediction to user management"
  - "integrate AI assistant for enterprise app"
  - "configure user management with ML service"
  - "deploy user management system with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application combining user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication (JWT), user CRUD operations
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: AI-powered classification, routing, and priority assignment
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, alerts

**Tech Stack**: React.js (frontend), Node.js/Express (backend), FastAPI (ML service), MongoDB (database), scikit-learn/River (ML)

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
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
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Server runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
DB_URI=mongodb://localhost:27017/enterprise-ums
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
# App runs at http://localhost:3000
```

## Key API Endpoints

### Authentication

```javascript
// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const { token, user } = await response.json();
  localStorage.setItem('token', token);
  return user;
};

// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(userData)
  });
  return await response.json();
};
```

### User Management

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
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
  return await response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
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
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
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
      category: ticketData.category, // 'technical', 'billing', 'general'
      priority: ticketData.priority
    })
  });
  return await response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${params}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## AI/ML Service API

### Ticket Classification

```python
# Python ML Service endpoint
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class TicketData(BaseModel):
    title: str
    description: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify ticket category and priority using ML"""
    # Feature extraction and prediction
    features = extract_features(ticket.title, ticket.description)
    category = category_model.predict([features])[0]
    priority = priority_model.predict([features])[0]
    
    return {
        "category": category,
        "priority": priority,
        "confidence": float(category_model.predict_proba([features]).max())
    }
```

```javascript
// Frontend usage
const classifyTicket = async (title, description) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title, description })
  });
  return await response.json();
};

// Use in ticket creation
const handleTicketSubmit = async (formData) => {
  const classification = await classifyTicket(formData.title, formData.description);
  const ticket = {
    ...formData,
    category: classification.category,
    priority: classification.priority
  };
  await createTicket(ticket, token);
};
```

### Risk Prediction

```python
@app.post("/api/ml/predict-risk")
async def predict_risk(user_data: dict):
    """Predict user risk score based on behavior patterns"""
    features = [
        user_data.get('failed_login_attempts', 0),
        user_data.get('avg_session_duration', 0),
        user_data.get('unusual_activity_count', 0),
        user_data.get('data_access_frequency', 0)
    ]
    
    risk_score = risk_model.predict([features])[0]
    risk_level = 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low'
    
    return {
        "risk_score": float(risk_score),
        "risk_level": risk_level,
        "factors": analyze_risk_factors(features)
    }
```

```javascript
// Check user risk
const checkUserRisk = async (userId, token) => {
  const response = await fetch(`http://localhost:8000/api/ml/predict-risk`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ userId })
  });
  return await response.json();
};
```

### Burnout Detection

```python
@app.post("/api/ml/detect-burnout")
async def detect_burnout(workload_data: dict):
    """Detect employee burnout risk from workload metrics"""
    features = [
        workload_data.get('tasks_completed', 0),
        workload_data.get('overtime_hours', 0),
        workload_data.get('missed_deadlines', 0),
        workload_data.get('avg_task_completion_time', 0),
        workload_data.get('ticket_response_time', 0)
    ]
    
    burnout_score = burnout_model.predict([features])[0]
    
    return {
        "burnout_score": float(burnout_score),
        "risk_level": categorize_burnout(burnout_score),
        "recommendations": get_burnout_recommendations(burnout_score)
    }
```

```javascript
// Monitor employee burnout
const checkBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ userId })
  });
  const result = await response.json();
  
  if (result.risk_level === 'high') {
    // Alert admin
    await sendBurnoutAlert(userId, result);
  }
  
  return result;
};
```

### Anomaly Detection

```python
from river import anomaly

# Online learning anomaly detector
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(activity_data: dict):
    """Detect anomalous user behavior in real-time"""
    features = {
        'login_time': activity_data.get('login_hour', 0),
        'access_count': activity_data.get('access_count', 0),
        'data_volume': activity_data.get('data_volume', 0),
        'location_change': activity_data.get('location_change', 0)
    }
    
    score = anomaly_detector.score_one(features)
    is_anomaly = score > threshold
    
    # Update model with new data
    anomaly_detector.learn_one(features)
    
    return {
        "is_anomaly": bool(is_anomaly),
        "anomaly_score": float(score),
        "timestamp": activity_data.get('timestamp')
    }
```

## Common Patterns

### Protected Routes with JWT

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No authentication token' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Usage in routes
app.get('/api/users', authMiddleware, async (req, res) => {
  // Protected route
});
```

### Role-Based Access Control

```javascript
// middleware/rbac.js
const checkRole = (allowedRoles) => {
  return (req, res, next) => {
    if (!req.user || !allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Access denied' });
    }
    next();
  };
};

// Usage
app.delete('/api/users/:id', 
  authMiddleware, 
  checkRole(['admin']), 
  async (req, res) => {
    // Admin-only route
  }
);
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId, token }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  
  useEffect(() => {
    loadTasks();
  }, [userId]);
  
  const loadTasks = async () => {
    const allTasks = await getUserTasks(userId, token);
    setTasks({
      todo: allTasks.filter(t => t.status === 'todo'),
      inProgress: allTasks.filter(t => t.status === 'in-progress'),
      done: allTasks.filter(t => t.status === 'done')
    });
  };
  
  const handleDrop = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus, token);
    loadTasks();
  };
  
  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} status="todo" onDrop={handleDrop} />
      <Column title="In Progress" tasks={tasks.inProgress} status="in-progress" onDrop={handleDrop} />
      <Column title="Done" tasks={tasks.done} status="done" onDrop={handleDrop} />
    </div>
  );
};
```

### Analytics Dashboard

```javascript
// components/AdminDashboard.jsx
const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  
  useEffect(() => {
    fetchAnalytics();
  }, []);
  
  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const [users, tasks, tickets, risks] = await Promise.all([
      fetch('http://localhost:5000/api/analytics/users', {
        headers: { 'Authorization': `Bearer ${token}` }
      }).then(r => r.json()),
      fetch('http://localhost:5000/api/analytics/tasks', {
        headers: { 'Authorization': `Bearer ${token}` }
      }).then(r => r.json()),
      fetch('http://localhost:5000/api/analytics/tickets', {
        headers: { 'Authorization': `Bearer ${token}` }
      }).then(r => r.json()),
      fetch('http://localhost:8000/api/ml/risk-summary', {
        headers: { 'Authorization': `Bearer ${token}` }
      }).then(r => r.json())
    ]);
    
    setAnalytics({ users, tasks, tickets, risks });
  };
  
  return (
    <div className="dashboard">
      <MetricsCard title="Total Users" value={analytics?.users.total} />
      <MetricsCard title="Active Tasks" value={analytics?.tasks.active} />
      <MetricsCard title="Open Tickets" value={analytics?.tickets.open} />
      <MetricsCard title="High Risk Users" value={analytics?.risks.highRisk} />
      {/* Charts and visualizations */}
    </div>
  );
};
```

## Configuration

### MongoDB Schema Examples

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  failedLoginAttempts: { type: Number, default: 0 }
});

module.exports = mongoose.model('User', userSchema);

// models/Task.js
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  completedAt: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);

// models/Ticket.js
const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  category: { type: String, enum: ['technical', 'billing', 'general'] },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'] },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  aiClassified: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### ML Model Training

```python
# ml-service/train_models.py
import pickle
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import pandas as pd

def train_ticket_classifier():
    """Train ticket classification model"""
    # Load training data
    df = pd.read_csv('data/tickets.csv')
    
    X = df[['title_length', 'desc_length', 'keyword_count', 'urgency_score']]
    y_category = df['category']
    y_priority = df['priority']
    
    # Train category classifier
    category_model = RandomForestClassifier(n_estimators=100, random_state=42)
    category_model.fit(X, y_category)
    
    # Train priority classifier
    priority_model = RandomForestClassifier(n_estimators=100, random_state=42)
    priority_model.fit(X, y_priority)
    
    # Save models
    with open('models/category_model.pkl', 'wb') as f:
        pickle.dump(category_model, f)
    with open('models/priority_model.pkl', 'wb') as f:
        pickle.dump(priority_model, f)

if __name__ == '__main__':
    train_ticket_classifier()
```

## Troubleshooting

### JWT Token Expiration

```javascript
// utils/api.js
const fetchWithAuth = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  });
  
  if (response.status === 401) {
    // Token expired, redirect to login
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Session expired');
  }
  
  return response;
};
```

### MongoDB Connection Issues

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

### ML Service Not Responding

```python
# Check ML service health
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "models_loaded": {
            "category_classifier": category_model is not None,
            "priority_classifier": priority_model is not None,
            "risk_predictor": risk_model is not None
        }
    }
```

```bash
# Test ML service
curl http://localhost:8000/health
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

### Model Performance Degradation

```python
# Implement model retraining
from apscheduler.schedulers.background import BackgroundScheduler

def retrain_models():
    """Periodically retrain models with new data"""
    print("Retraining models...")
    train_ticket_classifier()
    print("Models retrained successfully")

scheduler = BackgroundScheduler()
scheduler.add_job(retrain_models, 'interval', days=7)
scheduler.start()
```

## Deployment

### Environment Variables (Production)

```bash
# backend/.env.production
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=${ML_SERVICE_URL}
NODE_ENV=production
CORS_ORIGIN=${FRONTEND_URL}

# frontend/.env.production
REACT_APP_API_URL=${API_URL}
REACT_APP_ML_SERVICE_URL=${ML_SERVICE_URL}

# ml-service/.env.production
MODEL_PATH=/app/models
DB_URI=${MONGODB_URI}
LOG_LEVEL=WARNING
```

### Docker Deployment (Optional)

```dockerfile
# Backend Dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 5000
CMD ["npm", "start"]

# ML Service Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
