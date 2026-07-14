---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and predictive insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I configure AI analytics for user management"
  - "integrate AI ticket classification in my system"
  - "set up JWT authentication for enterprise app"
  - "create a user dashboard with task tracking"
  - "implement burnout detection and risk prediction"
  - "build a Kanban board with time tracking"
  - "configure the ML service for predictive analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

Enterprise User Management System is a full-stack JavaScript application that combines user management, task tracking, and support ticketing with AI-powered analytics. It provides:

- **User & Admin Dashboards**: Role-based access control with JWT authentication
- **Task Management**: Kanban boards with time tracking and progress monitoring
- **Support System**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Real-time Insights**: Performance metrics, audit logs, and predictive alerts

The system uses React frontend, Node.js backend, MongoDB database, and FastAPI ML service with scikit-learn and River for online learning.

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB running locally or connection string
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=\${YOUR_JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

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
```

Frontend runs at `http://localhost:3000`

## Key API Endpoints

### Authentication

```javascript
// Register user
POST /api/auth/register
Body: {
  "name": "John Doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
Body: {
  "email": "john@example.com",
  "password": "SecurePass123"
}
Response: { "token": "jwt_token", "user": {...} }
```

### User Management

```javascript
// Get all users (admin only)
GET /api/users
Headers: { "Authorization": "Bearer jwt_token" }

// Update user
PUT /api/users/:userId
Headers: { "Authorization": "Bearer jwt_token" }
Body: {
  "name": "Updated Name",
  "role": "admin",
  "status": "active"
}

// Delete user (admin only)
DELETE /api/users/:userId
Headers: { "Authorization": "Bearer jwt_token" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer jwt_token" }
Body: {
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
Body: { "status": "in-progress" } // or "done"

// Track time
POST /api/tasks/:taskId/time
Body: {
  "duration": 3600, // seconds
  "startTime": "2026-04-15T10:00:00Z",
  "endTime": "2026-04-15T11:00:00Z"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer jwt_token" }
Body: {
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "medium",
  "category": "technical"
}

// AI classification (automatic routing)
POST /api/ml/classify-ticket
Body: {
  "title": "Login issue",
  "description": "Cannot access dashboard after password reset"
}
Response: {
  "category": "technical",
  "priority": "high",
  "suggestedAssignee": "supportTeamId"
}
```

### AI Analytics

```javascript
// Risk prediction
POST /api/ml/predict-risk
Body: {
  "userId": "user123",
  "recentActivity": [...],
  "taskMetrics": {...}
}
Response: {
  "riskScore": 0.75,
  "riskLevel": "high",
  "factors": ["delayed_tasks", "low_activity"]
}

// Burnout detection
POST /api/ml/detect-burnout
Body: {
  "userId": "user123",
  "workloadData": {
    "hoursWorked": 55,
    "tasksCompleted": 12,
    "overtimeHours": 15
  }
}
Response: {
  "burnoutScore": 0.68,
  "status": "at_risk",
  "recommendations": ["Reduce workload", "Schedule break"]
}

// Anomaly detection
POST /api/ml/detect-anomaly
Body: {
  "userId": "user123",
  "activityLog": [...]
}
Response: {
  "isAnomaly": true,
  "anomalyType": "unusual_login_pattern",
  "confidence": 0.89
}

// Project delay prediction
POST /api/ml/predict-delay
Body: {
  "projectId": "proj123",
  "completionRate": 0.45,
  "daysRemaining": 10,
  "teamVelocity": 0.5
}
Response: {
  "delayPrediction": true,
  "estimatedDelay": 5, // days
  "confidence": 0.82
}
```

## Frontend Components

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
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/auth/login`,
      { email, password }
    );
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

  return { user, loading, login, logout, isAdmin: user?.role === 'admin' };
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
      { status: newStatus }
    );
    fetchTasks();
  };

  const Column = ({ title, tasks, status }) => (
    <div className="kanban-column">
      <h3>{title} ({tasks.length})</h3>
      {tasks.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority ${task.priority}`}>{task.priority}</span>
          <div className="task-actions">
            {status !== 'done' && (
              <button onClick={() => moveTask(task._id, 
                status === 'todo' ? 'in-progress' : 'done')}>
                Move →
              </button>
            )}
          </div>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} status="todo" />
      <Column title="In Progress" tasks={tasks.inProgress} status="in-progress" />
      <Column title="Done" tasks={tasks.done} status="done" />
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracker Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const [startTime, setStartTime] = useState(null);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const seconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  };

  const startTimer = () => {
    setIsRunning(true);
    setStartTime(new Date().toISOString());
  };

  const stopTimer = async () => {
    setIsRunning(false);
    const endTime = new Date().toISOString();
    
    await axios.post(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
      {
        duration: seconds,
        startTime,
        endTime
      }
    );
    
    setSeconds(0);
    setStartTime(null);
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <button onClick={isRunning ? stopTimer : startTimer}>
        {isRunning ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};

export default TimeTracker;
```

## Backend Implementation

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  department: String,
  position: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  metrics: {
    tasksCompleted: { type: Number, default: 0 },
    averageTaskTime: { type: Number, default: 0 },
    productivityScore: { type: Number, default: 0 }
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

userSchema.methods.generateToken = function() {
  return jwt.sign(
    { id: this._id, role: this.role },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRE }
  );
};

module.exports = mongoose.model('User', userSchema);
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const User = require('../models/User');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ success: false, error: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { duration, startTime, endTime } = req.body;
    const task = await Task.findById(req.params.id);
    
    task.timeEntries.push({ duration, startTime, endTime, userId: req.user.id });
    task.totalTimeSpent += duration;
    
    await task.save();
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ success: false, error: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    res.status(401).json({ success: false, error: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        success: false, 
        error: 'Not authorized to access this route' 
      });
    }
    next();
  };
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise AI Analytics Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
risk_model = None
burnout_model = None

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str
    recentActivity: List[Dict]
    taskMetrics: Dict

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workloadData: Dict[str, float]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityLog: List[Dict]

class DelayPredictionRequest(BaseModel):
    projectId: str
    completionRate: float
    daysRemaining: int
    teamVelocity: float

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-powered ticket classification"""
    text = f"{request.title} {request.description}".lower()
    
    # Simple rule-based classification (replace with trained model)
    if any(word in text for word in ["login", "password", "access", "crash"]):
        category = "technical"
        priority = "high"
    elif any(word in text for word in ["feature", "enhancement", "suggestion"]):
        category = "feature_request"
        priority = "low"
    elif any(word in text for word in ["bug", "error", "broken"]):
        category = "bug"
        priority = "medium"
    else:
        category = "general"
        priority = "medium"
    
    return {
        "category": category,
        "priority": priority,
        "suggestedAssignee": f"team_{category}",
        "confidence": 0.85
    }

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior"""
    metrics = request.taskMetrics
    
    # Calculate risk factors
    delayed_tasks = metrics.get("delayedTasks", 0)
    completion_rate = metrics.get("completionRate", 1.0)
    activity_score = metrics.get("activityScore", 1.0)
    
    risk_score = (
        (delayed_tasks * 0.3) +
        ((1 - completion_rate) * 0.4) +
        ((1 - activity_score) * 0.3)
    )
    
    risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.6 else "high"
    
    factors = []
    if delayed_tasks > 3:
        factors.append("delayed_tasks")
    if completion_rate < 0.7:
        factors.append("low_completion_rate")
    if activity_score < 0.5:
        factors.append("low_activity")
    
    return {
        "riskScore": round(risk_score, 2),
        "riskLevel": risk_level,
        "factors": factors,
        "recommendations": get_risk_recommendations(risk_level)
    }

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect employee burnout risk"""
    workload = request.workloadData
    
    hours_worked = workload.get("hoursWorked", 40)
    tasks_completed = workload.get("tasksCompleted", 10)
    overtime_hours = workload.get("overtimeHours", 0)
    
    # Burnout score calculation
    burnout_score = (
        (min(hours_worked / 60, 1.0) * 0.4) +
        (min(overtime_hours / 20, 1.0) * 0.4) +
        (max(0, 1 - tasks_completed / 15) * 0.2)
    )
    
    status = "healthy" if burnout_score < 0.3 else "at_risk" if burnout_score < 0.6 else "critical"
    
    recommendations = []
    if hours_worked > 50:
        recommendations.append("Reduce working hours")
    if overtime_hours > 10:
        recommendations.append("Schedule mandatory break")
    if tasks_completed > 20:
        recommendations.append("Delegate tasks")
    
    return {
        "burnoutScore": round(burnout_score, 2),
        "status": status,
        "recommendations": recommendations
    }

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    activity_log = request.activityLog
    
    if not activity_log:
        return {"isAnomaly": False, "anomalyType": None, "confidence": 0.0}
    
    # Check for unusual patterns
    login_times = [log.get("timestamp") for log in activity_log if log.get("action") == "login"]
    
    # Simple anomaly detection (replace with trained model)
    is_anomaly = len(login_times) > 10  # Too many logins
    
    return {
        "isAnomaly": is_anomaly,
        "anomalyType": "unusual_login_pattern" if is_anomaly else None,
        "confidence": 0.89 if is_anomaly else 0.0
    }

@app.post("/predict-delay")
async def predict_delay(request: DelayPredictionRequest):
    """Predict project delivery delay"""
    completion_rate = request.completionRate
    days_remaining = request.daysRemaining
    velocity = request.teamVelocity
    
    expected_completion = completion_rate + (velocity * days_remaining)
    
    delay_prediction = expected_completion < 1.0
    estimated_delay = max(0, int((1.0 - expected_completion) / velocity)) if delay_prediction else 0
    
    return {
        "delayPrediction": delay_prediction,
        "estimatedDelay": estimated_delay,
        "confidence": 0.82,
        "recommendations": [
            "Increase team capacity",
            "Reduce scope",
            "Extend deadline"
        ] if delay_prediction else []
    }

def get_risk_recommendations(risk_level: str) -> List[str]:
    if risk_level == "high":
        return ["Schedule 1-on-1 meeting", "Review workload", "Provide support"]
    elif risk_level == "medium":
        return ["Monitor progress", "Check-in weekly"]
    else:
        return ["Continue regular monitoring"]

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_secure_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
NODE_ENV=development
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENV=development
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
CACHE_ENABLED=true
```

## Common Patterns

### Role-Based Access Control

```javascript
// Protect admin routes
import { useAuth } from './hooks/useAuth';
import { Navigate } from 'react-router-dom';

const AdminRoute = ({ children }) => {
  const { user, loading } = useAuth();
  
  if (loading) return <div>Loading...</div>;
  if (!user || user.role !== 'admin') return <Navigate to="/dashboard" />;
  
  return children;
};

// Usage
<Route path="/admin" element={<AdminRoute><AdminPanel /></AdminRoute>} />
```

### Real-time Notifications

```javascript
// src/utils/notifications.js
import axios from 'axios';

export const subscribeToNotifications = (userId, callback) => {
  const eventSource = new EventSource(
    `${process.env.REACT_APP_API_URL}/notifications/stream/${userId}`
  );
  
  eventSource.onmessage = (event) => {
    const notification = JSON.parse(event.data);
    callback(notification);
  };
  
  return () => eventSource.close();
};

// Usage in component
useEffect(() => {
  const unsubscribe = subscribeToNotifications(user.id, (notification) => {
    toast.info(notification.message);
  });
  
  return unsubscribe;
}, [user.id]);
```

### AI Assistant Integration

```javascript
// src/components/AIAssistant.jsx
import React, { useState } from 'react';
import axios from 'axios';

const AIAssistant = () => {
  const [query, setQuery] = useState('');
  const [response, setResponse] = useState('');

  const askAI = async () => {
    const res = await axios.post(
      `${process.env.REACT_APP_ML_URL}/assistant/query`,
      { query, context: 'user_dashboard' }
    );
    setResponse(res.data.answer);
  };

  return (
    <div className="ai-assistant">
      <input 
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Ask AI anything..."
      />
      <button onClick={askAI}>Ask</button>
      {response && <div className="response">{response}</div>}
    </div>
  );
};
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add axios interceptor to handle token refresh
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
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Errors

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Responding

```python
# Check ML service health
import requests

response = requests.get("http://localhost:8000/health")
print(response.json())

# If 502 error, check if uvicorn is running
# uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Performance Issues with Large Datasets

```javascript
// Implement pagination
const getTasks = async (page = 1, limit = 20) => {
  const response = await axios.get(
    `${process.env.REACT_APP_API_URL}/tasks?page=${page}&limit=${limit}`
  );
  return response.data;
};

// Add indexing in MongoDB
db.tasks.createIndex({ assignedTo: 1, status: 1 });
db.users.createIndex({ email: 1 }, { unique: true });
```

### AI Model Accuracy Issues

```python
# Retrain model with new data
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

def retrain_risk_model(data):
    X = data.drop('risk_label', axis=1)
    y = data['risk_label']
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    
    model = RandomForestClassifier(n_estimators=100)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    print(f"Model accuracy: {accuracy}")
    
    joblib.dump(model, f"{MODEL_PATH}/risk_model.pkl")
```
