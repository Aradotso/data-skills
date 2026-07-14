---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with user roles"
  - "add ticket classification with machine learning"
  - "build kanban board with task tracking"
  - "integrate burnout detection and anomaly detection"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application for managing organizational users, tasks, and support tickets with integrated AI capabilities. It provides:

- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban board system with time tracking and progress monitoring
- **Ticket System**: Smart support ticket classification and routing using AI
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Centralized monitoring with audit logs and alerts

The system consists of three main components:
1. **Frontend**: React.js application
2. **Backend**: Node.js REST API with MongoDB
3. **ML Service**: FastAPI-based machine learning microservice

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Setup

```bash
# Clone repository
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
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017/enterprise_management
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
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

# Start frontend
npm start
```

## Key Components and APIs

### Backend API Endpoints

#### Authentication

```javascript
// Register new user
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
  "user": { "id", "name", "email", "role" }
}

// Get current user
GET /api/auth/me
Headers: { "Authorization": "Bearer <token>" }
```

#### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
Body: {
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
Body: { "name": "Jane Doe", "status": "active" }

// Delete user
DELETE /api/users/:id
```

#### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
Body: {
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:id
Body: { "status": "in-progress" }

// Track time
POST /api/tasks/:id/time
Body: { "timeSpent": 3600 } // seconds
```

#### Ticket Management

```javascript
// Create support ticket
POST /api/tickets
Body: {
  "title": "System not responding",
  "description": "Detailed issue description",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets
Query: ?status=open&priority=high

// Update ticket
PATCH /api/tickets/:id
Body: { "status": "resolved", "resolution": "Fixed the bug" }
```

### ML Service API Endpoints

#### Ticket Classification

```python
# Classify ticket using AI
POST /api/ml/classify-ticket
Body: {
  "title": "Cannot access dashboard",
  "description": "Getting 500 error when logging in"
}
Response: {
  "category": "technical",
  "priority": "high",
  "confidence": 0.89,
  "suggested_assignee": "tech_support"
}
```

#### Risk Detection

```python
# Predict user risk score
POST /api/ml/risk-detection
Body: {
  "user_id": "user_id_here",
  "failed_logins": 3,
  "unusual_activity": true,
  "last_login": "2026-04-15T10:30:00Z"
}
Response: {
  "risk_score": 0.75,
  "risk_level": "high",
  "factors": ["multiple_failed_logins", "unusual_access_pattern"],
  "recommendations": ["require_2fa", "password_reset"]
}
```

#### Burnout Analysis

```python
# Analyze user burnout risk
POST /api/ml/burnout-detection
Body: {
  "user_id": "user_id_here",
  "tasks_count": 25,
  "avg_hours_per_day": 11,
  "missed_deadlines": 3,
  "overtime_days": 14
}
Response: {
  "burnout_score": 0.82,
  "burnout_risk": "high",
  "factors": ["excessive_workload", "overtime", "missed_deadlines"],
  "recommendations": ["reduce_workload", "vacation", "redistribute_tasks"]
}
```

#### Project Insights

```python
# Predict project delay risk
POST /api/ml/project-insights
Body: {
  "project_id": "proj_123",
  "tasks_total": 50,
  "tasks_completed": 20,
  "days_elapsed": 30,
  "deadline_days": 60
}
Response: {
  "delay_probability": 0.68,
  "predicted_completion": "2026-06-15",
  "bottlenecks": ["resource_shortage", "dependency_delays"],
  "recommendations": ["add_resources", "prioritize_critical_path"]
}
```

#### Anomaly Detection

```python
# Detect anomalies in user behavior
POST /api/ml/anomaly-detection
Body: {
  "user_id": "user_id_here",
  "login_time": "03:45:00",
  "ip_address": "192.168.1.100",
  "location": "Unusual City",
  "actions": ["bulk_download", "admin_access_attempt"]
}
Response: {
  "is_anomaly": true,
  "anomaly_score": 0.91,
  "detected_patterns": ["unusual_time", "suspicious_location", "bulk_operations"],
  "severity": "critical"
}
```

## Frontend Usage Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
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
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);

  const loadUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (err) {
      console.error('Auth error:', err);
      logout();
    }
    setLoading(false);
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
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
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
    const categorized = {
      todo: res.data.filter(t => t.status === 'todo'),
      inProgress: res.data.filter(t => t.status === 'in-progress'),
      done: res.data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
      status: newStatus
    });
    fetchTasks();
  };

  const renderColumn = (title, status, taskList) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {taskList.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <div className="task-actions">
            {status !== 'in-progress' && (
              <button onClick={() => moveTask(task._id, 'in-progress')}>
                Start
              </button>
            )}
            {status !== 'done' && (
              <button onClick={() => moveTask(task._id, 'done')}>
                Complete
              </button>
            )}
          </div>
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
// src/pages/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [aiInsights, setAiInsights] = useState({});
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    // Fetch users
    const usersRes = await axios.get(`${process.env.REACT_APP_API_URL}/api/users`);
    setUsers(usersRes.data);

    // Fetch AI insights for each user
    const insights = {};
    for (const user of usersRes.data) {
      try {
        const riskRes = await axios.post(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`,
          { user_id: user._id }
        );
        const burnoutRes = await axios.post(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`,
          { user_id: user._id }
        );
        insights[user._id] = {
          risk: riskRes.data,
          burnout: burnoutRes.data
        };
      } catch (err) {
        console.error(`Error fetching insights for ${user._id}:`, err);
      }
    }
    setAiInsights(insights);

    // Generate alerts
    const newAlerts = usersRes.data
      .filter(user => {
        const userInsights = insights[user._id];
        return userInsights?.risk?.risk_level === 'high' ||
               userInsights?.burnout?.burnout_risk === 'high';
      })
      .map(user => ({
        userId: user._id,
        userName: user.name,
        message: `${user.name} requires attention`,
        insights: insights[user._id]
      }));
    setAlerts(newAlerts);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {/* Alerts Section */}
      {alerts.length > 0 && (
        <div className="alerts-section">
          <h2>Critical Alerts</h2>
          {alerts.map(alert => (
            <div key={alert.userId} className="alert-card">
              <h3>{alert.userName}</h3>
              {alert.insights.risk?.risk_level === 'high' && (
                <p className="risk-alert">High security risk detected</p>
              )}
              {alert.insights.burnout?.burnout_risk === 'high' && (
                <p className="burnout-alert">High burnout risk detected</p>
              )}
            </div>
          ))}
        </div>
      )}

      {/* Users Table */}
      <div className="users-section">
        <h2>Users Overview</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Risk Score</th>
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
                <td className={aiInsights[user._id]?.risk?.risk_level}>
                  {aiInsights[user._id]?.risk?.risk_score?.toFixed(2) || 'N/A'}
                </td>
                <td className={aiInsights[user._id]?.burnout?.burnout_risk}>
                  {aiInsights[user._id]?.burnout?.burnout_score?.toFixed(2) || 'N/A'}
                </td>
                <td>
                  <button onClick={() => viewUserDetails(user._id)}>View</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Backend Implementation Patterns

### JWT Middleware

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
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized` 
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.getTasks = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort('-createdAt');
    
    res.json(tasks);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });
    res.status(201).json(task);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Check authorization
    if (req.user.role !== 'admin' && 
        task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    Object.assign(task, req.body);
    await task.save();
    
    res.json(task);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.timeSpent = (task.timeSpent || 0) + req.body.timeSpent;
    task.timeEntries = task.timeEntries || [];
    task.timeEntries.push({
      user: req.user._id,
      duration: req.body.timeSpent,
      timestamp: new Date()
    });
    
    await task.save();
    res.json(task);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
};
```

## ML Service Implementation

### Ticket Classification Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import pickle
import os

class TicketClassifier:
    def __init__(self, model_path='./models/ticket_classifier.pkl'):
        self.model_path = model_path
        self.pipeline = None
        self.load_or_create_model()
    
    def load_or_create_model(self):
        if os.path.exists(self.model_path):
            with open(self.model_path, 'rb') as f:
                self.pipeline = pickle.load(f)
        else:
            self.pipeline = Pipeline([
                ('tfidf', TfidfVectorizer(max_features=1000)),
                ('clf', MultinomialNB())
            ])
    
    def train(self, texts, labels):
        self.pipeline.fit(texts, labels)
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.pipeline, f)
    
    def predict(self, text):
        if self.pipeline is None:
            raise ValueError("Model not trained")
        
        combined_text = f"{text.get('title', '')} {text.get('description', '')}"
        prediction = self.pipeline.predict([combined_text])[0]
        probabilities = self.pipeline.predict_proba([combined_text])[0]
        confidence = max(probabilities)
        
        return {
            'category': prediction,
            'confidence': float(confidence)
        }

# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.ticket_classifier import TicketClassifier
from models.risk_detector import RiskDetector
from models.burnout_analyzer import BurnoutAnalyzer
import uvicorn

app = FastAPI(title="Enterprise ML Service")

# Initialize models
ticket_classifier = TicketClassifier()
risk_detector = RiskDetector()
burnout_analyzer = BurnoutAnalyzer()

class TicketInput(BaseModel):
    title: str
    description: str

class RiskInput(BaseModel):
    user_id: str
    failed_logins: int = 0
    unusual_activity: bool = False
    last_login: str

class BurnoutInput(BaseModel):
    user_id: str
    tasks_count: int
    avg_hours_per_day: float
    missed_deadlines: int
    overtime_days: int

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    try:
        result = ticket_classifier.predict({
            'title': ticket.title,
            'description': ticket.description
        })
        
        # Determine priority based on keywords
        priority = 'medium'
        urgent_keywords = ['urgent', 'critical', 'down', 'broken', 'not working']
        if any(word in ticket.title.lower() or word in ticket.description.lower() 
               for word in urgent_keywords):
            priority = 'high'
        
        return {
            **result,
            'priority': priority,
            'suggested_assignee': 'tech_support' if result['category'] == 'technical' else 'general_support'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/risk-detection")
async def detect_risk(data: RiskInput):
    try:
        risk_score = risk_detector.calculate_risk(data.dict())
        
        risk_level = 'low'
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        
        factors = []
        if data.failed_logins > 2:
            factors.append('multiple_failed_logins')
        if data.unusual_activity:
            factors.append('unusual_access_pattern')
        
        recommendations = []
        if risk_level == 'high':
            recommendations.extend(['require_2fa', 'password_reset', 'review_access'])
        
        return {
            'risk_score': risk_score,
            'risk_level': risk_level,
            'factors': factors,
            'recommendations': recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(data: BurnoutInput):
    try:
        burnout_score = burnout_analyzer.analyze(data.dict())
        
        burnout_risk = 'low'
        if burnout_score > 0.7:
            burnout_risk = 'high'
        elif burnout_score > 0.4:
            burnout_risk = 'medium'
        
        factors = []
        if data.avg_hours_per_day > 10:
            factors.append('excessive_workload')
        if data.overtime_days > 10:
            factors.append('overtime')
        if data.missed_deadlines > 2:
            factors.append('missed_deadlines')
        
        recommendations = []
        if burnout_risk == 'high':
            recommendations.extend(['reduce_workload', 'vacation', 'redistribute_tasks'])
        
        return {
            'burnout_score': burnout_score,
            'burnout_risk': burnout_risk,
            'factors': factors,
            'recommendations': recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Risk Detection Model

```python
# ml-service/models/risk_detector.py
import numpy as np
from datetime import datetime

class RiskDetector:
    def __init__(self):
        self.weights = {
            'failed_logins': 0.3,
            'unusual_activity': 0.4,
            'time_anomaly': 0.3
        }
    
    def calculate_risk(self, data):
        risk_score = 0.0
        
        # Failed login attempts
        failed_logins = data.get('failed_logins', 0)
        if failed_logins > 0:
            risk_score += min(failed_logins / 5, 1.0) * self.weights['failed_logins']
        
        # Unusual activity flag
        if data.get('unusual_activity', False):
            risk_score += self.weights['unusual_activity']
        
        # Time-based anomaly
        last_login = data.get('last_login')
        if last_login:
            try:
                login_time = datetime.fromisoformat(last_login.replace('Z', '+00:00'))
                hour = login_time.hour
                if hour < 6 or hour > 22:  # Outside normal hours
                    risk_score += self.weights['time_anomaly']
            except:
                pass
        
        return min(risk_score, 1.0)
```

### Burnout Analyzer

```python
# ml-service/models/burnout_analyzer.py
class BurnoutAnalyzer:
    def __init__(self):
        self.thresholds = {
            'max_healthy_hours': 8,
            'max_healthy_tasks': 15,
            'max_overtime_days': 5,
            'acceptable_missed_deadlines': 1
        }
    
    def analyze(self, data):
        burnout_score = 0.0
        
        # Workload analysis
        avg_hours = data.get('avg_hours_per_day', 8)
        if avg_hours > self.thresholds['max_healthy_hours']:
            hours_factor = (avg_hours - self.thresholds['max_healthy_hours']) / 4
            burnout_score += min(hours_factor * 0.3, 0.3)
        
        # Task count
        tasks_count = data.get('tasks_count', 0)
        if tasks_count > self.thresholds['max_healthy_tasks']:
            tasks_factor = (tasks_count - self.thresholds['max_healthy_tasks']) / 10
            burnout_score += min(tasks_factor * 0.25, 0.25)
        
        # Overtime days
        overtime_days = data.get('overtime_days', 0)
        if overtime_days > self.thresholds['max_overtime_days']:
            overtime_factor = (overtime_days - self.thresholds['max_overtime_days']) / 10
            burnout_score += min(overtime_factor * 0.25, 0.25)
        
        # Missed deadlines
        missed_deadlines = data.get('missed_deadlines', 0)
        if missed_deadlines > self.thresholds['acceptable_missed_deadlines']:
            deadline_factor = missed_deadlines / 5
            burnout_score += min(deadline_factor * 0.2, 0.2)
        
        return min(burnout_score, 1.0)
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=your_secure_random_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MODEL_PATH=./models
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017/enterprise_management
MAX_WORKERS=4
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Common Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### ML Model Not Found

```python
# Initialize models with fallback
try:
    ticket_classifier = TicketClassifier()
except FileNotFoundError:
    print("Training new ticket classifier...")
    ticket_classifier = TicketClassifier()
