---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with ML"
  - "build task tracking system with AI insights"
  - "configure AI ticket classification system"
  - "integrate anomaly detection for users"
  - "set up burnout prediction analytics"
  - "deploy enterprise user management with FastAPI ML"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application combining React frontend, Node.js backend, and FastAPI ML service to manage users, tasks, and support tickets with AI-powered insights. The system provides risk detection, anomaly detection, burnout analysis, and predictive project insights using machine learning models.

**Key capabilities:**
- User authentication and role-based access control (JWT)
- Task management with Kanban board (To Do → In Progress → Done)
- AI-based ticket classification and smart routing
- Risk prediction and anomaly detection
- Burnout detection using workload analysis
- Predictive project insights and delay forecasting

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or remote connection string

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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

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

npm start
```

Frontend runs at `http://localhost:3000`

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   React     │────▶│  Node.js    │────▶│   MongoDB   │
│  Frontend   │     │   Backend   │     │  Database   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   FastAPI   │
                    │  ML Service │
                    └─────────────┘
```

## Backend API Endpoints

### Authentication

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'user' or 'admin'
    })
  });
  const data = await response.json();
  return data.token;
};

// POST /api/auth/login
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
```

### User Management (Admin)

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
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
  return await response.json();
};

// DELETE /api/users/:id - Delete user (Admin only)
const deleteUser = async (userId, token) => {
  await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
};
```

### Task Management

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
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'in-progress', 'done'
  });
  return await response.json();
};
```

### Ticket Management

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
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category, // 'technical', 'access', 'general'
      priority: ticketData.priority
    })
  });
  return await response.json();
};

// GET /api/tickets - Get all tickets (Admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// PATCH /api/tickets/:id - Update ticket status
const updateTicketStatus = async (ticketId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'open', 'in-progress', 'resolved'
  });
  return await response.json();
};
```

## ML Service API

### AI Ticket Classification

```python
# FastAPI endpoint for ticket classification
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib

app = FastAPI()

class TicketInput(BaseModel):
    title: str
    description: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    """
    Classify ticket into categories: technical, access, general
    Returns category and confidence score
    """
    text = f"{ticket.title} {ticket.description}"
    
    # Load trained model
    model = joblib.load('./models/ticket_classifier.pkl')
    vectorizer = joblib.load('./models/vectorizer.pkl')
    
    # Predict
    X = vectorizer.transform([text])
    category = model.predict(X)[0]
    confidence = model.predict_proba(X).max()
    
    return {
        "category": category,
        "confidence": float(confidence),
        "recommended_priority": "high" if confidence > 0.8 else "medium"
    }
```

**Usage from Node.js backend:**

```javascript
const classifyTicket = async (title, description) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title, description })
  });
  return await response.json();
};
```

### Risk Prediction

```python
from river import linear_model, preprocessing
from river import compose

class RiskPredictor(BaseModel):
    user_id: str
    failed_logins: int
    tasks_overdue: int
    avg_task_completion_time: float
    last_activity_hours: float

@app.post("/api/ml/predict-risk")
async def predict_risk(data: RiskPredictor):
    """
    Predict user risk score based on behavior patterns
    Returns risk level: low, medium, high
    """
    # Online learning model (River)
    model = compose.Pipeline(
        preprocessing.StandardScaler(),
        linear_model.LogisticRegression()
    )
    
    features = {
        'failed_logins': data.failed_logins,
        'tasks_overdue': data.tasks_overdue,
        'avg_completion_time': data.avg_task_completion_time,
        'inactivity_hours': data.last_activity_hours
    }
    
    risk_score = model.predict_proba_one(features).get(1, 0)
    
    if risk_score > 0.7:
        risk_level = "high"
    elif risk_score > 0.4:
        risk_level = "medium"
    else:
        risk_level = "low"
    
    return {
        "user_id": data.user_id,
        "risk_score": float(risk_score),
        "risk_level": risk_level,
        "recommendations": get_risk_recommendations(risk_level)
    }

def get_risk_recommendations(level):
    if level == "high":
        return ["Review account activity", "Contact user", "Temporary access restriction"]
    elif level == "medium":
        return ["Monitor activity", "Send reminder notification"]
    return ["No action needed"]
```

### Burnout Detection

```python
class BurnoutInput(BaseModel):
    user_id: str
    tasks_completed_week: int
    avg_hours_per_day: float
    missed_deadlines: int
    weekend_work_hours: float
    stress_indicators: int

@app.post("/api/ml/detect-burnout")
async def detect_burnout(data: BurnoutInput):
    """
    Detect employee burnout risk using workload analysis
    """
    # Calculate burnout score
    workload_score = min(data.avg_hours_per_day / 8.0, 1.5)
    stress_score = (data.missed_deadlines * 0.1 + data.stress_indicators * 0.15)
    weekend_penalty = data.weekend_work_hours * 0.2
    
    burnout_score = (workload_score + stress_score + weekend_penalty) / 3
    
    if burnout_score > 0.7:
        level = "critical"
        actions = ["Immediate intervention", "Reduce workload", "Schedule 1-on-1"]
    elif burnout_score > 0.5:
        level = "warning"
        actions = ["Monitor closely", "Consider workload adjustment"]
    else:
        level = "healthy"
        actions = ["Continue monitoring"]
    
    return {
        "user_id": data.user_id,
        "burnout_score": float(burnout_score),
        "level": level,
        "recommended_actions": actions,
        "metrics": {
            "workload_factor": float(workload_score),
            "stress_factor": float(stress_score),
            "weekend_work_impact": float(weekend_penalty)
        }
    }
```

### Anomaly Detection

```python
from river import anomaly
import numpy as np

class AnomalyInput(BaseModel):
    user_id: str
    login_time_hour: int
    location_changed: bool
    failed_attempts: int
    data_access_count: int
    unusual_api_calls: int

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(data: AnomalyInput):
    """
    Detect anomalous user behavior for security
    """
    # Half-Space Trees for anomaly detection
    model = anomaly.HalfSpaceTrees(seed=42)
    
    features = {
        'login_hour': data.login_time_hour,
        'location_change': int(data.location_changed),
        'failed_attempts': data.failed_attempts,
        'data_access': data.data_access_count,
        'api_calls': data.unusual_api_calls
    }
    
    anomaly_score = model.score_one(features)
    model.learn_one(features)
    
    is_anomaly = anomaly_score > 0.6
    
    return {
        "user_id": data.user_id,
        "is_anomaly": is_anomaly,
        "anomaly_score": float(anomaly_score),
        "severity": "high" if anomaly_score > 0.8 else "medium" if anomaly_score > 0.6 else "low",
        "flags": get_anomaly_flags(data, anomaly_score)
    }

def get_anomaly_flags(data, score):
    flags = []
    if data.failed_attempts > 3:
        flags.append("Multiple failed login attempts")
    if data.location_changed:
        flags.append("Location change detected")
    if data.unusual_api_calls > 10:
        flags.append("Unusual API activity")
    return flags
```

## Frontend Integration

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [burnoutData, setBurnoutData] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUserTasks();
    fetchBurnoutAnalysis();
  }, []);

  const fetchUserTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/user/me`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const fetchBurnoutAnalysis = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/burnout`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setBurnoutData(response.data);
    } catch (error) {
      console.error('Failed to fetch burnout data:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="dashboard">
      {burnoutData && burnoutData.level !== 'healthy' && (
        <div className={`alert alert-${burnoutData.level}`}>
          Burnout Warning: {burnoutData.level}
        </div>
      )}
      
      <div className="kanban-board">
        <TaskColumn title="To Do" tasks={tasks.todo} onMove={moveTask} />
        <TaskColumn title="In Progress" tasks={tasks.inProgress} onMove={moveTask} />
        <TaskColumn title="Done" tasks={tasks.done} onMove={moveTask} />
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAlerts, setRiskAlerts] = useState([]);
  const [anomalies, setAnomalies] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUsers();
    fetchRiskAlerts();
    fetchAnomalies();
  }, []);

  const fetchUsers = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/users`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setUsers(response.data);
  };

  const fetchRiskAlerts = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/analytics/risk-alerts`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setRiskAlerts(response.data);
  };

  const fetchAnomalies = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/analytics/anomalies`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setAnomalies(response.data);
  };

  const analyzeUserRisk = async (userId) => {
    const response = await axios.post(
      `${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`,
      {
        user_id: userId,
        failed_logins: 2,
        tasks_overdue: 3,
        avg_task_completion_time: 5.5,
        last_activity_hours: 72
      }
    );
    
    alert(`Risk Level: ${response.data.risk_level}\nScore: ${response.data.risk_score}`);
  };

  return (
    <div className="admin-dashboard">
      <section className="alerts">
        <h2>Security Alerts</h2>
        {anomalies.map(anomaly => (
          <div key={anomaly.id} className="alert-card">
            <span className="severity">{anomaly.severity}</span>
            <p>{anomaly.user_id}: {anomaly.flags.join(', ')}</p>
          </div>
        ))}
      </section>

      <section className="risk-overview">
        <h2>Risk Alerts</h2>
        {riskAlerts.map(alert => (
          <div key={alert.user_id} className="risk-card">
            <h3>{alert.user_name}</h3>
            <p>Risk Level: {alert.risk_level}</p>
            <button onClick={() => analyzeUserRisk(alert.user_id)}>
              Re-analyze
            </button>
          </div>
        ))}
      </section>

      <section className="user-management">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.isActive ? 'Active' : 'Inactive'}</td>
                <td>
                  <button onClick={() => analyzeUserRisk(user._id)}>
                    Check Risk
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Backend Environment Variables

```bash
# .env in backend/
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

### ML Service Environment Variables

```bash
# .env in ml-service/
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
RETRAIN_INTERVAL=86400
MIN_TRAINING_SAMPLES=100
```

### Frontend Environment Variables

```bash
# .env in frontend/
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_VERSION=1.0.0
```

## Common Patterns

### Protected Route with JWT

```javascript
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user'));

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requireAdmin && user?.role !== 'admin') {
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

### Real-time Task Timer

```javascript
const TaskTimer = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const saveTimeLog = async () => {
    const token = localStorage.getItem('token');
    await axios.post(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
      { duration: seconds },
      { headers: { Authorization: `Bearer ${token}` } }
    );
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const seconds = secs % 60;
    return `${hours}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="timer">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTimeLog} disabled={seconds === 0}>
        Save
      </button>
    </div>
  );
};
```

### Automated Ticket Classification

```javascript
const createTicketWithAI = async (ticketData) => {
  const token = localStorage.getItem('token');
  
  // Step 1: Classify with AI
  const classification = await axios.post(
    `${process.env.REACT_APP_ML_URL}/api/ml/classify-ticket`,
    {
      title: ticketData.title,
      description: ticketData.description
    }
  );

  // Step 2: Create ticket with AI-determined category
  const ticket = await axios.post(
    `${process.env.REACT_APP_API_URL}/tickets`,
    {
      ...ticketData,
      category: classification.data.category,
      priority: classification.data.recommended_priority,
      ai_confidence: classification.data.confidence
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );

  return ticket.data;
};
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

**Common fixes:**
- Ensure MongoDB is running: `sudo systemctl start mongod`
- Check connection string format
- Verify network access if using MongoDB Atlas

### JWT Token Expiration

```javascript
// Add axios interceptor for token refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Model Not Loading

```python
# ml-service/main.py
import os
from pathlib import Path

@app.on_event("startup")
async def load_models():
    model_path = Path(os.getenv('MODEL_PATH', './models'))
    model_path.mkdir(exist_ok=True)
    
    try:
        # Load or initialize models
        if (model_path / 'ticket_classifier.pkl').exists():
            global ticket_model
            ticket_model = joblib.load(model_path / 'ticket_classifier.pkl')
            print("✓ Ticket classifier loaded")
        else:
            print("⚠ No pre-trained model found, will train on first request")
    except Exception as e:
        print(f"Model loading error: {e}")
```

### CORS Errors

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
}));
```

### Performance Issues with Large Datasets

```javascript
// Implement pagination for user list
const getUsersPaginated = async (page = 1, limit = 20) => {
  const response = await axios.get(
    `${process.env.REACT_APP_API_URL}/users?page=${page}&limit=${limit}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Backend implementation
router.get('/users', auth, async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;

  const users = await User.find()
    .skip(skip)
    .limit(limit)
    .select('-password');
  
  const total = await User.countDocuments();
  
  res.json({
    users,
    currentPage: page,
    totalPages: Math.ceil(total / limit),
    totalUsers: total
  });
});
```

## Production Deployment

### Environment-specific configurations

```javascript
// frontend/src/config.js
const config = {
  development: {
    apiUrl: 'http://localhost:5000/api',
    mlUrl: 'http://localhost:8000'
  },
  production: {
    apiUrl: process.env.REACT_APP_API_URL,
    mlUrl: process.env.REACT_APP_ML_URL
  }
};

export default config[process.env.NODE_ENV || 'development'];
```

### Docker Compose Setup

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:5.0
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise_user_mgmt
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise_user_mgmt
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  mongo-data:
```

Run with: `docker-compose up -d`
