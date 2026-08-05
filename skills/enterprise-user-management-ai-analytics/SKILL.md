---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create user dashboard with task tracking"
  - "implement AI risk detection and anomaly detection"
  - "build admin panel with user management"
  - "add JWT authentication to user system"
  - "configure ML service for predictive analytics"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-driven features including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Tech Stack:**
- Frontend: React.js
- Backend: Node.js with Express
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Authentication: JWT

## Installation

### Prerequisites
- Node.js (v14+)
- Python 3.8+
- MongoDB instance

### Clone and Setup

```bash
# Clone the repository
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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend server
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
DATABASE_URL=${MONGODB_URI}
MODEL_PATH=./models
EOF

# Start ML service
uvicorn main:app --reload --port 8000
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

## Project Architecture

```
├── frontend/           # React application
│   ├── src/
│   │   ├── components/  # Reusable UI components
│   │   ├── pages/       # Dashboard, Login, Admin pages
│   │   ├── services/    # API integration
│   │   └── utils/       # Helper functions
├── backend/            # Node.js Express API
│   ├── models/         # MongoDB schemas
│   ├── routes/         # API endpoints
│   ├── middleware/     # Auth, validation
│   └── controllers/    # Business logic
└── ml-service/         # FastAPI ML microservice
    ├── models/         # ML model files
    ├── services/       # AI logic
    └── main.py         # FastAPI app
```

## Backend API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
// User login
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
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users
// Fetch all users (admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id
// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
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

// DELETE /api/users/:id
// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management

```javascript
// POST /api/tasks
// Create new task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
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

// GET /api/tasks/user/:userId
// Get tasks for specific user
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status
// Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets
// Get all tickets
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### AI-Powered Analytics

```python
# ML Service Example: Risk Prediction
# File: ml-service/services/risk_prediction.py

from sklearn.ensemble import RandomForestClassifier
import numpy as np

class RiskPredictor:
    def __init__(self):
        self.model = RandomForestClassifier()
        self.trained = False
    
    def predict_risk(self, user_data):
        """
        Predict risk level based on user behavior
        """
        features = np.array([[
            user_data['failed_logins'],
            user_data['late_submissions'],
            user_data['inactive_days'],
            user_data['task_overdue_count']
        ]])
        
        if not self.trained:
            return {'risk_level': 'unknown', 'confidence': 0}
        
        prediction = self.model.predict(features)[0]
        probability = self.model.predict_proba(features)[0]
        
        return {
            'risk_level': 'high' if prediction == 1 else 'low',
            'confidence': float(max(probability))
        }
```

### Risk Detection API

```javascript
// POST /api/ml/risk-prediction
// Predict user risk
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ user_id: userId })
  });
  return response.json();
  // Returns: { risk_level: 'high', confidence: 0.85, factors: [...] }
};
```

### Anomaly Detection

```python
# File: ml-service/services/anomaly_detection.py

from river import anomaly
from river import preprocessing

class AnomalyDetector:
    def __init__(self):
        self.scaler = preprocessing.StandardScaler()
        self.detector = anomaly.HalfSpaceTrees()
    
    def detect_anomaly(self, activity_data):
        """
        Detect anomalous user behavior
        """
        features = {
            'login_hour': activity_data['login_hour'],
            'login_location': activity_data['login_location_code'],
            'actions_per_minute': activity_data['actions_per_minute'],
            'failed_attempts': activity_data['failed_attempts']
        }
        
        scaled = self.scaler.transform_one(features)
        score = self.detector.score_one(scaled)
        self.detector.learn_one(scaled)
        
        return {
            'is_anomaly': score > 0.7,
            'anomaly_score': score,
            'timestamp': activity_data['timestamp']
        }
```

```javascript
// POST /api/ml/anomaly-detection
// Detect anomalous activity
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(activityData)
  });
  return response.json();
};
```

### Burnout Analysis

```javascript
// POST /api/ml/burnout-analysis
// Analyze employee burnout risk
const analyzeBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ user_id: userId })
  });
  return response.json();
  // Returns: { burnout_risk: 'medium', score: 0.65, recommendations: [...] }
};
```

### Predictive Insights

```javascript
// POST /api/ml/project-prediction
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      tasks_completed: projectData.completed,
      tasks_remaining: projectData.remaining,
      avg_completion_time: projectData.avgTime,
      deadline: projectData.deadline
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.75, estimated_completion: '2026-05-01' }
};
```

### AI Ticket Classification

```python
# File: ml-service/services/ticket_classifier.py

from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB

class TicketClassifier:
    def __init__(self):
        self.vectorizer = TfidfVectorizer()
        self.classifier = MultinomialNB()
        self.categories = ['Technical', 'HR', 'General', 'Urgent']
    
    def classify_ticket(self, ticket_text):
        """
        Classify support ticket and suggest routing
        """
        features = self.vectorizer.transform([ticket_text])
        prediction = self.classifier.predict(features)[0]
        probabilities = self.classifier.predict_proba(features)[0]
        
        return {
            'category': self.categories[prediction],
            'confidence': float(max(probabilities)),
            'suggested_department': self._route_to_department(prediction)
        }
    
    def _route_to_department(self, category_idx):
        routing = {
            0: 'IT Support',
            1: 'Human Resources',
            2: 'General Admin',
            3: 'Emergency Response'
        }
        return routing.get(category_idx, 'General Admin')
```

```javascript
// POST /api/ml/classify-ticket
// Classify and route ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text: ticketText })
  });
  return response.json();
};
```

## Frontend Components

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutScore, setBurnoutScore] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    const token = localStorage.getItem('token');
    const userId = localStorage.getItem('userId');
    
    try {
      // Fetch tasks
      const tasksRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setTasks(tasksRes.data);

      // Fetch burnout analysis
      const burnoutRes = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`,
        { user_id: userId }
      );
      setBurnoutScore(burnoutRes.data);
      
      setLoading(false);
    } catch (error) {
      console.error('Error fetching data:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData(); // Refresh data
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      {/* Burnout Alert */}
      {burnoutScore && burnoutScore.burnout_risk !== 'low' && (
        <div className="alert">
          <h3>⚠️ Burnout Risk: {burnoutScore.burnout_risk}</h3>
          <p>Score: {burnoutScore.score}</p>
        </div>
      )}

      {/* Kanban Board */}
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do</h3>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => updateTaskStatus(task.id, 'in-progress')}>
                Start Task
              </button>
            </div>
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress</h3>
          {tasks.filter(t => t.status === 'in-progress').map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => updateTaskStatus(task.id, 'done')}>
                Complete
              </button>
            </div>
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>Done</h3>
          {tasks.filter(t => t.status === 'done').map(task => (
            <div key={task.id} className="task-card completed">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskUsers, setRiskUsers] = useState([]);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    const token = localStorage.getItem('token');
    
    try {
      // Fetch all users
      const usersRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setUsers(usersRes.data);

      // Analyze each user for risk
      const riskPromises = usersRes.data.map(user =>
        axios.post(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`,
          { user_id: user.id }
        )
      );
      const riskResults = await Promise.all(riskPromises);
      const highRiskUsers = riskResults
        .map((res, idx) => ({ ...usersRes.data[idx], risk: res.data }))
        .filter(u => u.risk.risk_level === 'high');
      setRiskUsers(highRiskUsers);

    } catch (error) {
      console.error('Error fetching admin data:', error);
    }
  };

  const deleteUser = async (userId) => {
    const token = localStorage.getItem('token');
    if (window.confirm('Are you sure you want to delete this user?')) {
      try {
        await axios.delete(
          `${process.env.REACT_APP_API_URL}/api/users/${userId}`,
          { headers: { Authorization: `Bearer ${token}` } }
        );
        fetchAdminData(); // Refresh data
      } catch (error) {
        console.error('Error deleting user:', error);
      }
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>

      {/* Risk Alerts */}
      {riskUsers.length > 0 && (
        <div className="risk-section">
          <h2>⚠️ High Risk Users</h2>
          {riskUsers.map(user => (
            <div key={user.id} className="risk-card">
              <h3>{user.username}</h3>
              <p>Risk Level: {user.risk.risk_level}</p>
              <p>Confidence: {(user.risk.confidence * 100).toFixed(0)}%</p>
            </div>
          ))}
        </div>
      )}

      {/* User Management Table */}
      <div className="users-table">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user.id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.status}</td>
                <td>
                  <button onClick={() => deleteUser(user.id)}>Delete</button>
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

## Common Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, requireAdmin };
```

### MongoDB User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  failedLogins: {
    type: Number,
    default: 0
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
  this.password = await bcrypt.hash(this.password, 10);
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
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
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
    default: 0 // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### FastAPI ML Service Main App

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# Enable CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Request models
class RiskPredictionRequest(BaseModel):
    user_id: str

class AnomalyDetectionRequest(BaseModel):
    login_hour: int
    login_location_code: int
    actions_per_minute: float
    failed_attempts: int
    timestamp: str

class BurnoutAnalysisRequest(BaseModel):
    user_id: str

class TicketClassificationRequest(BaseModel):
    text: str

# Initialize ML services
from services.risk_prediction import RiskPredictor
from services.anomaly_detection import AnomalyDetector
from services.ticket_classifier import TicketClassifier

risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
ticket_classifier = TicketClassifier()

@app.get("/")
def read_root():
    return {"message": "Enterprise AI Analytics Service", "version": "1.0"}

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Fetch user data from database
        user_data = get_user_metrics(request.user_id)
        result = risk_predictor.predict_risk(user_data)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        result = anomaly_detector.detect_anomaly(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        user_data = get_user_workload(request.user_id)
        # Calculate burnout score based on workload
        score = calculate_burnout_score(user_data)
        risk_level = 'high' if score > 0.7 else 'medium' if score > 0.4 else 'low'
        
        return {
            'burnout_risk': risk_level,
            'score': score,
            'recommendations': get_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify_ticket(request.text)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Helper functions
def get_user_metrics(user_id):
    # Fetch from MongoDB
    return {
        'failed_logins': 2,
        'late_submissions': 5,
        'inactive_days': 3,
        'task_overdue_count': 4
    }

def get_user_workload(user_id):
    # Fetch task data
    return {
        'active_tasks': 15,
        'overdue_tasks': 3,
        'avg_daily_hours': 9.5,
        'weekend_work_hours': 8
    }

def calculate_burnout_score(data):
    # Weighted calculation
    score = (
        (data['active_tasks'] / 20) * 0.3 +
        (data['overdue_tasks'] / 10) * 0.2 +
        (data['avg_daily_hours'] / 12) * 0.3 +
        (data['weekend_work_hours'] / 16) * 0.2
    )
    return min(score, 1.0)

def get_recommendations(risk_level):
    recommendations = {
        'high': [
            'Reduce workload immediately',
            'Schedule vacation days',
            'Delegate tasks to team members',
            'Consider mental health support'
        ],
        'medium': [
            'Review task priorities',
            'Improve time management',
            'Take regular breaks',
            'Discuss workload with manager'
        ],
        'low': [
            'Maintain current balance',
            'Continue good practices',
            'Stay mindful of workload changes'
        ]
    }
    return recommendations.get(risk_level, [])

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Environment Variables

**Backend (.env):**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRATION=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env):**
```env
DATABASE_URL=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
ENABLE_MODEL_CACHING=true
```

**Frontend (.env):**
```env
REACT_APP_
