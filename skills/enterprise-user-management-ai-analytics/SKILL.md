---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and workforce insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics for user management"
  - "show me how to create user dashboards with task tracking"
  - "implement ticket classification and risk detection"
  - "build a kanban board with time tracking"
  - "configure JWT authentication for user management"
  - "set up AI-powered burnout detection and anomaly alerts"
  - "create an admin dashboard with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides:

- **User Management**: Role-based access control, authentication, and user profiles
- **Task Management**: Kanban boards, time tracking, and task assignment
- **Ticket System**: Support ticket creation, tracking, and AI-based classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Centralized monitoring, audit logs, and organizational analytics

**Tech Stack**: React.js frontend, Node.js backend, FastAPI ML service, MongoDB database, JWT authentication

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip (for ML service)
python --version
pip --version

# MongoDB running locally or connection string ready
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables

# ML Service setup
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env

# Frontend setup
cd ../frontend
npm install
cp .env.example .env
```

### Environment Configuration

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```bash
PORT=8000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securePass123",
  "role": "user"  // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securePass123"
}
// Returns: { token, user: { id, name, email, role } }

// Get current user
GET /api/auth/me
Headers: { Authorization: "Bearer <JWT_TOKEN>" }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <JWT_TOKEN>" }

// Create user
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@company.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "name": "Jane Smith Updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { Authorization: "Bearer <JWT_TOKEN>" }

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new analytics dashboard",
  "assignedTo": "userId123",
  "status": "todo",  // todo, in_progress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in_progress"
}

// Track time
POST /api/tasks/:taskId/time
{
  "duration": 3600,  // seconds
  "date": "2026-04-15"
}
```

### Ticket Management

```javascript
// Create support ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "category": "technical",
  "priority": "high"
}

// Get tickets
GET /api/tickets?status=open&priority=high

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Password reset link was expired. Generated new link."
}
```

## ML Service API Reference

### AI Ticket Classification

```python
# FastAPI endpoint for ticket classification
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class TicketRequest(BaseModel):
    title: str
    description: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    """
    Classifies ticket into categories: technical, billing, access, general
    """
    # Simple keyword-based classification (replace with ML model)
    text = f"{ticket.title} {ticket.description}".lower()
    
    categories = {
        "technical": ["error", "bug", "crash", "not working", "login"],
        "billing": ["payment", "invoice", "charge", "subscription"],
        "access": ["permission", "access", "role", "cannot view"],
        "general": []
    }
    
    for category, keywords in categories.items():
        if any(keyword in text for keyword in keywords):
            return {
                "category": category,
                "confidence": 0.85,
                "suggested_priority": "high" if category == "technical" else "medium"
            }
    
    return {"category": "general", "confidence": 0.60, "suggested_priority": "low"}
```

### Risk Prediction

```python
from sklearn.ensemble import RandomForestClassifier
import numpy as np

class RiskPredictor:
    def __init__(self):
        self.model = RandomForestClassifier(n_estimators=100)
        
    def predict_user_risk(self, user_features):
        """
        Predicts risk level based on user behavior
        Features: login_frequency, failed_logins, after_hours_access, 
                 data_download_volume, permission_changes
        """
        # Example features
        features = np.array([[
            user_features.get('login_frequency', 5),
            user_features.get('failed_logins', 0),
            user_features.get('after_hours_access', 2),
            user_features.get('data_download_volume', 100),
            user_features.get('permission_changes', 1)
        ]])
        
        # Mock prediction (train model with real data)
        risk_score = np.random.uniform(0, 1)
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": self._identify_risk_factors(user_features)
        }
    
    def _identify_risk_factors(self, features):
        factors = []
        if features.get('failed_logins', 0) > 5:
            factors.append("High failed login attempts")
        if features.get('after_hours_access', 0) > 10:
            factors.append("Excessive after-hours access")
        if features.get('data_download_volume', 0) > 1000:
            factors.append("Unusual data download volume")
        return factors

@app.post("/api/ml/predict-risk")
async def predict_risk(user_id: str, features: dict):
    predictor = RiskPredictor()
    return predictor.predict_user_risk(features)
```

### Burnout Detection

```python
from river import anomaly
from river import preprocessing

class BurnoutDetector:
    def __init__(self):
        # Online learning model for detecting anomalies
        self.model = anomaly.HalfSpaceTrees(seed=42)
        self.scaler = preprocessing.StandardScaler()
        
    def detect_burnout(self, workload_data):
        """
        Detects potential burnout based on workload metrics
        Metrics: tasks_completed, avg_work_hours, overtime_hours, 
                task_completion_rate, stress_indicators
        """
        features = {
            'tasks_completed': workload_data.get('tasks_completed', 0),
            'avg_work_hours': workload_data.get('avg_work_hours', 8),
            'overtime_hours': workload_data.get('overtime_hours', 0),
            'task_completion_rate': workload_data.get('task_completion_rate', 0.9),
            'missed_deadlines': workload_data.get('missed_deadlines', 0)
        }
        
        # Burnout indicators
        burnout_score = 0
        warnings = []
        
        if features['overtime_hours'] > 20:
            burnout_score += 0.3
            warnings.append("Excessive overtime hours")
            
        if features['avg_work_hours'] > 10:
            burnout_score += 0.25
            warnings.append("Long average work hours")
            
        if features['task_completion_rate'] < 0.6:
            burnout_score += 0.2
            warnings.append("Low task completion rate")
            
        if features['missed_deadlines'] > 5:
            burnout_score += 0.25
            warnings.append("High number of missed deadlines")
        
        burnout_level = "critical" if burnout_score > 0.7 else "warning" if burnout_score > 0.4 else "normal"
        
        return {
            "burnout_score": round(burnout_score, 2),
            "burnout_level": burnout_level,
            "warnings": warnings,
            "recommendations": self._get_recommendations(burnout_level)
        }
    
    def _get_recommendations(self, level):
        if level == "critical":
            return ["Immediate workload reduction needed", "Schedule time off", "Reassign tasks"]
        elif level == "warning":
            return ["Monitor workload closely", "Encourage breaks", "Review task priorities"]
        return ["Maintain current pace", "Continue regular check-ins"]

@app.post("/api/ml/detect-burnout")
async def detect_burnout(user_id: str, workload: dict):
    detector = BurnoutDetector()
    return detector.detect_burnout(workload)
```

### Predictive Project Insights

```python
@app.post("/api/ml/predict-delay")
async def predict_project_delay(project_data: dict):
    """
    Predicts if project will be delayed based on current metrics
    """
    tasks_total = project_data.get('tasks_total', 0)
    tasks_completed = project_data.get('tasks_completed', 0)
    days_remaining = project_data.get('days_remaining', 30)
    avg_completion_rate = project_data.get('avg_completion_rate', 0)
    
    tasks_remaining = tasks_total - tasks_completed
    required_rate = tasks_remaining / days_remaining if days_remaining > 0 else float('inf')
    
    delay_probability = min(required_rate / (avg_completion_rate + 0.01), 1.0)
    
    return {
        "delay_probability": round(delay_probability, 2),
        "status": "on_track" if delay_probability < 0.3 else "at_risk" if delay_probability < 0.7 else "likely_delayed",
        "tasks_remaining": tasks_remaining,
        "required_daily_rate": round(required_rate, 2),
        "current_daily_rate": round(avg_completion_rate, 2),
        "estimated_delay_days": max(0, int((tasks_remaining / avg_completion_rate) - days_remaining)) if avg_completion_rate > 0 else 0
    }
```

## Frontend Implementation

### React Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
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
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
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

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      const taskData = response.data;
      
      setTasks({
        todo: taskData.filter(t => t.status === 'todo'),
        in_progress: taskData.filter(t => t.status === 'in_progress'),
        done: taskData.filter(t => t.status === 'done')
      });
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };
  
  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };
  
  const handleDragOver = (e) => {
    e.preventDefault();
  };
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div className="kanban-board">
      {Object.entries(tasks).map(([status, taskList]) => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          <div className="task-list">
            {taskList.map(task => (
              <div
                key={task._id}
                className={`task-card priority-${task.priority}`}
                draggable
                onDragStart={(e) => handleDragStart(e, task._id)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <span className="task-priority">{task.priority}</span>
                {task.dueDate && (
                  <span className="task-due">Due: {new Date(task.dueDate).toLocaleDateString()}</span>
                )}
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { LineChart, Line, BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutAlerts: [],
    projectDelays: []
  });
  
  useEffect(() => {
    fetchAnalytics();
  }, []);
  
  const fetchAnalytics = async () => {
    try {
      // Fetch risk predictions
      const riskResponse = await axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-analysis`);
      
      // Fetch burnout data
      const burnoutResponse = await axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`);
      
      // Fetch project delays
      const delayResponse = await axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/project-delays`);
      
      setAnalytics({
        riskUsers: riskResponse.data,
        burnoutAlerts: burnoutResponse.data,
        projectDelays: delayResponse.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };
  
  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Analytics</h2>
      
      <div className="analytics-grid">
        {/* Risk Detection */}
        <div className="analytics-card">
          <h3>User Risk Detection</h3>
          <div className="risk-list">
            {analytics.riskUsers.map(user => (
              <div key={user.id} className={`risk-item risk-${user.risk_level}`}>
                <span>{user.name}</span>
                <span className="risk-score">Risk: {user.risk_score}</span>
                <ul className="risk-factors">
                  {user.factors.map((factor, idx) => (
                    <li key={idx}>{factor}</li>
                  ))}
                </ul>
              </div>
            ))}
          </div>
        </div>
        
        {/* Burnout Alerts */}
        <div className="analytics-card">
          <h3>Burnout Alerts</h3>
          <BarChart width={400} height={300} data={analytics.burnoutAlerts}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="name" />
            <YAxis />
            <Tooltip />
            <Legend />
            <Bar dataKey="burnout_score" fill="#ff6b6b" />
          </BarChart>
        </div>
        
        {/* Project Delay Predictions */}
        <div className="analytics-card">
          <h3>Project Delay Predictions</h3>
          {analytics.projectDelays.map(project => (
            <div key={project.id} className={`project-status status-${project.status}`}>
              <h4>{project.name}</h4>
              <p>Delay Probability: {(project.delay_probability * 100).toFixed(0)}%</p>
              <p>Estimated Delay: {project.estimated_delay_days} days</p>
              <div className="progress-bar">
                <div
                  className="progress-fill"
                  style={{ width: `${(project.tasks_completed / project.tasks_total) * 100}%` }}
                />
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
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
    let interval = null;
    
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else if (interval) {
      clearInterval(interval);
    }
    
    return () => clearInterval(interval);
  }, [isRunning]);
  
  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };
  
  const saveTime = async () => {
    try {
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
        { duration: seconds, date: new Date().toISOString() }
      );
      setSeconds(0);
      setIsRunning(false);
      alert('Time logged successfully!');
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };
  
  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <div className="time-controls">
        <button onClick={() => setIsRunning(!isRunning)}>
          {isRunning ? 'Pause' : 'Start'}
        </button>
        <button onClick={() => setSeconds(0)} disabled={isRunning}>
          Reset
        </button>
        <button onClick={saveTime} disabled={seconds === 0 || isRunning}>
          Save Time
        </button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();
  
  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### API Service Layer

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

export const authService = {
  login: (email, password) => axios.post(`${API_URL}/api/auth/login`, { email, password }),
  register: (data) => axios.post(`${API_URL}/api/auth/register`, data),
  getCurrentUser: () => axios.get(`${API_URL}/api/auth/me`)
};

export const taskService = {
  getTasks: () => axios.get(`${API_URL}/api/tasks`),
  createTask: (data) => axios.post(`${API_URL}/api/tasks`, data),
  updateTask: (id, data) => axios.put(`${API_URL}/api/tasks/${id}`, data),
  deleteTask: (id) => axios.delete(`${API_URL}/api/tasks/${id}`),
  logTime: (id, duration) => axios.post(`${API_URL}/api/tasks/${id}/time`, { duration })
};

export const ticketService = {
  getTickets: (params) => axios.get(`${API_URL}/api/tickets`, { params }),
  createTicket: (data) => axios.post(`${API_URL}/api/tickets`, data),
  classifyTicket: (title, description) => 
    axios.post(`${ML_API_URL}/api/ml/classify-ticket`, { title, description })
};

export const mlService = {
  predictRisk: (userId, features) => 
    axios.post(`${ML_API_URL}/api/ml/predict-risk`, { user_id: userId, features }),
  detectBurnout: (userId, workload) => 
    axios.post(`${ML_API_URL}/api/ml/detect-burnout`, { user_id: userId, workload }),
  predictDelay: (projectData) => 
    axios.post(`${ML_API_URL}/api/ml/predict-delay`, projectData)
};
```

## Troubleshooting

### Common Issues

**MongoDB Connection Failed**
```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Or use Docker
docker run -d -p 27017:27017 --name mongodb mongo:latest
```

**JWT Token Expired**
```javascript
// Add axios interceptor to handle token refresh
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

**CORS Errors**
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**ML Service Not Responding**
```bash
# Check if FastAPI is running
curl http://localhost:8000/docs

# Check Python dependencies
pip list | grep -E 'fastapi|scikit-learn|river'

# Reinstall if needed
pip install --upgrade fastapi scikit-learn river uvicorn
```

**Model Performance Issues**
```python
# Use model caching to improve response time
from functools import lru_cache

@lru_cache(maxsize=100)
def get_model():
    return load_pretrained_model()

# Implement async processing for heavy computations
from fastapi import BackgroundTasks

@app.post("/api/ml/heavy-analysis")
async def heavy_analysis(data: dict, background_tasks: BackgroundTasks):
    background_tasks.add_task(process_analysis, data)
    return {"status": "processing"}
```

**Database Query Performance**
```javascript
// Add indexes to MongoDB collections
db.tasks.createIndex({ assignedTo: 1, status: 1 })
db.users.createIndex({ email: 1 }, { unique: true })
db.tickets.createIndex({ createdAt: -1, status: 1 })
```

## Security Best Practices

```javascript
// Password hashing (backend)
const bcrypt = require('bcryptjs');

const hashPassword = async (password) => {
  const salt = await bcrypt.genSalt(10);
  return await bcrypt.hash(password, salt);
};

// Input validation
const { body, validationResult } = require('express-validator');

app.post('/api/users', [
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }),
  body('name').trim().notEmpty()
], async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  // Process request
});

// Rate limiting
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to assist developers in implementing, configuring, and extending the system effectively.
