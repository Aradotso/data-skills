---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with task tracking"
  - "implement AI-powered ticket classification system"
  - "build kanban board with user authentication"
  - "add burnout detection and risk prediction"
  - "configure enterprise task management with ML insights"
  - "integrate AI assistant for user management"
  - "deploy user management system with anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack web application that combines user management, task tracking, and support ticket systems with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What It Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban-style boards (To Do → In Progress → Done) with time tracking
- **Support System**: Ticket creation and AI-powered classification/routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Dashboards**: Admin analytics and user performance insights

**Tech Stack**: React.js frontend, Node.js backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

```bash
# Required installations
node --version  # v14+ required
python --version  # Python 3.8+ required
mongod --version  # MongoDB required
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
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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

Create `ml-service/.env`:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
API_KEY=your_ml_api_key_here
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_WS_URL=ws://localhost:5000
```

Start frontend:
```bash
npm start
# Runs on http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register - Register new user
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

// POST /api/auth/login - User login
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
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
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

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create task
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
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
  });
  return response.json();
};
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  const ticket = await response.json();
  
  // Auto-classify with AI
  await classifyTicket(ticket._id, token);
  return ticket;
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### AI/ML Endpoints (ML Service)

```javascript
// POST /classify-ticket - AI ticket classification
const classifyTicket = async (ticketId, token) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      ticket_id: ticketId,
      text: "Ticket description or subject",
      priority: "medium"
    })
  });
  return response.json();
  // Returns: { category: 'technical', confidence: 0.89, route_to: 'IT_Team' }
};

// POST /predict-risk - User risk prediction
const predictUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: 45,
      task_completion_rate: 0.65,
      avg_response_time: 120,
      failed_logins: 3
    })
  });
  return response.json();
  // Returns: { risk_score: 0.72, risk_level: 'medium', factors: [...] }
};

// POST /detect-anomaly - Anomaly detection
const detectAnomaly = async (userData, token) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userData.userId,
      login_time: new Date().toISOString(),
      location: userData.location,
      device: userData.device,
      activity_pattern: userData.activityPattern
    })
  });
  return response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, alert: true }
};

// POST /burnout-analysis - Burnout detection
const analyzeBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/burnout-analysis', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      tasks_count: 15,
      avg_hours_worked: 52,
      overtime_hours: 12,
      stress_level: 7,
      task_completion_rate: 0.88
    })
  });
  return response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
};

// POST /predict-delay - Project delay prediction
const predictProjectDelay = async (projectData, token) => {
  const response = await fetch('http://localhost:8000/predict-delay', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      project_id: projectData.projectId,
      tasks_total: 50,
      tasks_completed: 30,
      days_remaining: 10,
      team_size: 5,
      complexity_score: 8
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 5, suggestions: [...] }
};
```

## React Component Patterns

### Protected Route with Authentication

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" replace />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" replace />;
  }

  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
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
```

### Admin Dashboard with AI Insights

```javascript
// src/pages/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({});
  const [risks, setRisks] = useState([]);
  const [anomalies, setAnomalies] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    // Get organization analytics
    const analyticsRes = await fetch('http://localhost:5000/api/analytics/overview', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setAnalytics(await analyticsRes.json());

    // Get high-risk users
    const risksRes = await fetch('http://localhost:5000/api/analytics/risks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setRisks(await risksRes.json());

    // Get recent anomalies
    const anomaliesRes = await fetch('http://localhost:8000/recent-anomalies', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setAnomalies(await anomaliesRes.json());
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <StatCard title="Total Users" value={analytics.totalUsers} />
        <StatCard title="Active Tasks" value={analytics.activeTasks} />
        <StatCard title="Open Tickets" value={analytics.openTickets} />
        <StatCard title="Completion Rate" value={`${analytics.completionRate}%`} />
      </div>

      <div className="alerts-section">
        <h2>High Risk Users</h2>
        {risks.map(risk => (
          <RiskAlert key={risk.userId} data={risk} />
        ))}
      </div>

      <div className="anomalies-section">
        <h2>Security Anomalies</h2>
        {anomalies.map(anomaly => (
          <AnomalyAlert key={anomaly.id} data={anomaly} />
        ))}
      </div>
    </div>
  );
};
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const startTimer = () => setIsRunning(true);
  
  const stopTimer = async () => {
    setIsRunning(false);
    // Log time to backend
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: { 
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ duration: seconds })
    });
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
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
```

## Backend Model Examples

### User Model (MongoDB)

```javascript
// backend/models/User.js
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
  loginAttempts: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'], default: 'medium' },
  category: String, // Set by AI classification
  aiClassified: { type: Boolean, default: false },
  confidence: Number,
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service Implementation (Python)

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class TicketData(BaseModel):
    ticket_id: str
    text: str
    priority: str

class RiskData(BaseModel):
    user_id: str
    login_frequency: int
    task_completion_rate: float
    avg_response_time: int
    failed_logins: int

class AnomalyData(BaseModel):
    user_id: str
    login_time: str
    location: str
    device: str
    activity_pattern: list

class BurnoutData(BaseModel):
    user_id: str
    tasks_count: int
    avg_hours_worked: float
    overtime_hours: float
    stress_level: int
    task_completion_rate: float

@app.post("/classify-ticket")
async def classify_ticket(data: TicketData):
    """AI-powered ticket classification"""
    # Simple keyword-based classification (replace with trained model)
    categories = {
        'technical': ['bug', 'error', 'crash', 'not working', 'broken'],
        'account': ['login', 'password', 'access', 'permissions'],
        'billing': ['payment', 'invoice', 'charge', 'subscription'],
        'general': []
    }
    
    text_lower = data.text.lower()
    for category, keywords in categories.items():
        if any(keyword in text_lower for keyword in keywords):
            return {
                'category': category,
                'confidence': 0.85,
                'route_to': f'{category.upper()}_Team'
            }
    
    return {
        'category': 'general',
        'confidence': 0.60,
        'route_to': 'SUPPORT_Team'
    }

@app.post("/predict-risk")
async def predict_risk(data: RiskData):
    """Predict user risk score"""
    # Calculate risk based on multiple factors
    factors = []
    risk_score = 0.0
    
    # Login frequency check
    if data.login_frequency < 10:
        risk_score += 0.2
        factors.append("Low login frequency")
    
    # Task completion rate
    if data.task_completion_rate < 0.7:
        risk_score += 0.3
        factors.append("Low task completion rate")
    
    # Failed logins
    if data.failed_logins > 2:
        risk_score += 0.3
        factors.append("Multiple failed login attempts")
    
    # Response time
    if data.avg_response_time > 180:
        risk_score += 0.2
        factors.append("Slow response time")
    
    risk_level = 'low' if risk_score < 0.4 else 'medium' if risk_score < 0.7 else 'high'
    
    return {
        'risk_score': round(risk_score, 2),
        'risk_level': risk_level,
        'factors': factors,
        'user_id': data.user_id
    }

@app.post("/detect-anomaly")
async def detect_anomaly(data: AnomalyData):
    """Detect anomalous user behavior"""
    # Simple anomaly detection logic
    anomaly_score = 0.0
    is_anomaly = False
    
    # Check for unusual login times (e.g., 2-5 AM)
    hour = int(data.login_time.split('T')[1].split(':')[0])
    if 2 <= hour <= 5:
        anomaly_score += 0.4
    
    # Check for new location
    # In production, compare with user's typical locations
    if 'unusual' in data.location.lower():
        anomaly_score += 0.3
    
    # Check for new device
    if 'new' in data.device.lower():
        anomaly_score += 0.3
    
    is_anomaly = anomaly_score > 0.5
    
    return {
        'is_anomaly': is_anomaly,
        'anomaly_score': round(anomaly_score, 2),
        'alert': is_anomaly,
        'user_id': data.user_id,
        'timestamp': data.login_time
    }

@app.post("/burnout-analysis")
async def analyze_burnout(data: BurnoutData):
    """Analyze employee burnout risk"""
    burnout_score = 0.0
    recommendations = []
    
    # Work hours analysis
    if data.avg_hours_worked > 45:
        burnout_score += 0.3
        recommendations.append("Reduce weekly working hours")
    
    # Overtime analysis
    if data.overtime_hours > 10:
        burnout_score += 0.2
        recommendations.append("Limit overtime work")
    
    # Stress level
    if data.stress_level > 6:
        burnout_score += 0.3
        recommendations.append("Consider stress management resources")
    
    # Task load
    if data.tasks_count > 10 and data.task_completion_rate < 0.8:
        burnout_score += 0.2
        recommendations.append("Redistribute workload")
    
    risk = 'low' if burnout_score < 0.4 else 'medium' if burnout_score < 0.7 else 'high'
    
    return {
        'burnout_risk': risk,
        'score': round(burnout_score, 2),
        'recommendations': recommendations,
        'user_id': data.user_id
    }

@app.post("/predict-delay")
async def predict_delay(data: dict):
    """Predict project delay probability"""
    tasks_remaining = data['tasks_total'] - data['tasks_completed']
    completion_rate = data['tasks_completed'] / data['tasks_total']
    
    # Simple delay prediction
    required_rate = tasks_remaining / data['days_remaining']
    current_rate = data['tasks_completed'] / 30  # Assume 30 days elapsed
    
    delay_probability = 0.0
    if required_rate > current_rate * 1.2:
        delay_probability = 0.7
    elif required_rate > current_rate:
        delay_probability = 0.4
    else:
        delay_probability = 0.1
    
    estimated_delay = max(0, int((tasks_remaining / current_rate) - data['days_remaining']))
    
    suggestions = []
    if delay_probability > 0.5:
        suggestions.append("Increase team size")
        suggestions.append("Prioritize critical tasks")
        suggestions.append("Consider deadline extension")
    
    return {
        'delay_probability': round(delay_probability, 2),
        'estimated_delay_days': estimated_delay,
        'suggestions': suggestions,
        'project_id': data['project_id']
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Configuration

### Database Connection

```javascript
// backend/config/database.js
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

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization?.startsWith('Bearer')) {
    try {
      token = req.headers.authorization.split(' ')[1];
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = await User.findById(decoded.id).select('-password');
      next();
    } catch (error) {
      res.status(401).json({ message: 'Not authorized, token failed' });
    }
  }
  
  if (!token) {
    res.status(401).json({ message: 'Not authorized, no token' });
  }
};

const admin = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};

module.exports = { protect, admin };
```

## Common Workflows

### Complete User Registration Flow

```javascript
// Register → Login → Access Dashboard
