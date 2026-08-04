---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket routing, and behavioral insights
triggers:
  - "help me build a user management system with AI analytics"
  - "how do I set up enterprise user management with AI features"
  - "create task management with anomaly detection"
  - "implement AI-powered ticket classification system"
  - "build user dashboard with risk prediction"
  - "set up Kanban board with burnout detection"
  - "integrate AI analytics into user management app"
  - "deploy enterprise management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, ticket classification, and predictive insights.

## What This Project Does

This system provides:
- **User Management**: Role-based access control (Admin/User), authentication via JWT
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: Raise, track, and AI-classify support requests
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics and user performance tracking
- **Audit Logs**: Track all user activities and detect suspicious behavior

## Architecture

The project has three main components:
1. **Frontend** (React.js) - User interface on port 3000
2. **Backend** (Node.js) - REST API on port 5000
3. **ML Service** (FastAPI + scikit-learn) - AI analytics on port 8000

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or connection string

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

Create `.env` file in backend directory:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your_jwt_secret_key_here
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

Create `.env` file in ml-service directory:
```env
MODEL_PATH=./models
LOG_LEVEL=info
BACKEND_URL=http://localhost:5000
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Backend API Reference

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const response = await fetch('http://localhost:5000/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com',
    password: 'securePassword123',
    role: 'user' // or 'admin'
  })
});
const data = await response.json();
// Returns: { token, user: { id, name, email, role } }
```

**Login**
```javascript
// POST /api/auth/login
const response = await fetch('http://localhost:5000/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'john@example.com',
    password: 'securePassword123'
  })
});
const { token, user } = await response.json();
localStorage.setItem('authToken', token);
```

### User Management (Admin Only)

**Get All Users**
```javascript
// GET /api/users
const response = await fetch('http://localhost:5000/api/users', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`
  }
});
const users = await response.json();
```

**Update User**
```javascript
// PUT /api/users/:userId
const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'Updated Name',
    role: 'admin',
    status: 'active'
  })
});
```

**Delete User**
```javascript
// DELETE /api/users/:userId
await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'DELETE',
  headers: { 'Authorization': `Bearer ${token}` }
});
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
const response = await fetch('http://localhost:5000/api/tasks', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Implement new feature',
    description: 'Add AI analytics dashboard',
    assignedTo: 'userId123',
    priority: 'high', // low, medium, high
    dueDate: '2026-05-01',
    status: 'todo' // todo, in-progress, done
  })
});
const task = await response.json();
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
  headers: { 'Authorization': `Bearer ${token}` }
});
const tasks = await response.json();
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:taskId/status
await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ status: 'in-progress' })
});
```

**Track Time**
```javascript
// POST /api/tasks/:taskId/time
await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    duration: 3600, // seconds
    date: new Date().toISOString()
  })
});
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const response = await fetch('http://localhost:5000/api/tickets', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Login issue',
    description: 'Cannot login with correct credentials',
    priority: 'high',
    category: 'technical' // technical, billing, general
  })
});
const ticket = await response.json();
```

**Get All Tickets (Admin)**
```javascript
// GET /api/tickets
const response = await fetch('http://localhost:5000/api/tickets', {
  headers: { 'Authorization': `Bearer ${token}` }
});
const tickets = await response.json();
```

**Update Ticket Status**
```javascript
// PATCH /api/tickets/:ticketId
await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: 'resolved', // open, in-progress, resolved, closed
    resolution: 'Password reset performed'
  })
});
```

## ML Service API Reference

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    title: 'Cannot access dashboard',
    description: 'Getting 403 error when trying to access admin dashboard',
    userId: 'user123'
  })
});
const result = await response.json();
// Returns: { category: 'technical', priority: 'high', suggestedAssignee: 'adminId' }
```

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    loginAttempts: 5,
    lastLoginTime: '2026-04-15T10:30:00Z',
    tasksCompleted: 12,
    ticketsRaised: 8,
    averageTaskTime: 7200,
    accessPatterns: ['dashboard', 'users', 'admin', 'settings']
  })
});
const { riskScore, riskLevel, factors } = await response.json();
// riskLevel: 'low', 'medium', 'high'
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    activityData: {
      loginTime: '03:00', // unusual login time
      location: 'Unknown',
      deviceId: 'new-device-123',
      actionsPerMinute: 50, // unusually high
      dataAccessed: ['sensitive-user-data', 'financial-records']
    }
  })
});
const { isAnomaly, anomalyScore, reasons } = await response.json();
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    weeklyHours: 65,
    tasksAssigned: 25,
    tasksCompleted: 15,
    overdueTasksCount: 8,
    avgTaskCompletionTime: 14400,
    weekendWork: true,
    nightWorkHours: 15
  })
});
const { burnoutRisk, score, recommendations } = await response.json();
// burnoutRisk: 'low', 'moderate', 'high', 'critical'
```

### Project Delay Prediction

```javascript
// POST /api/ml/predict-delay
const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    projectId: 'proj123',
    totalTasks: 50,
    completedTasks: 20,
    overdueTasks: 5,
    teamSize: 8,
    daysRemaining: 30,
    avgTaskCompletionRate: 0.6,
    blockerCount: 3
  })
});
const { delayProbability, estimatedDelay, suggestions } = await response.json();
```

## Frontend Component Patterns

### Protected Route with Authentication

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('authToken');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  
  useEffect(() => {
    fetchTasks();
  }, [userId]);
  
  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      { headers: { 'Authorization': `Bearer ${localStorage.getItem('authToken')}` }}
    );
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };
  
  const moveTask = async (taskId, newStatus) => {
    await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      }
    );
    fetchTasks();
  };
  
  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => moveTask(task.id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Risk Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const AIRiskDashboard = () => {
  const [insights, setInsights] = useState({
    highRiskUsers: [],
    anomalies: [],
    burnoutRisks: [],
    projectDelays: []
  });
  
  useEffect(() => {
    fetchAIInsights();
  }, []);
  
  const fetchAIInsights = async () => {
    const token = localStorage.getItem('authToken');
    
    // Fetch high-risk users
    const riskResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/high-risk-users`,
      { headers: { 'Authorization': `Bearer ${token}` }}
    );
    const highRiskUsers = await riskResponse.json();
    
    // Fetch anomalies
    const anomalyResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/recent-anomalies`,
      { headers: { 'Authorization': `Bearer ${token}` }}
    );
    const anomalies = await anomalyResponse.json();
    
    setInsights({ highRiskUsers, anomalies });
  };
  
  return (
    <div className="ai-dashboard">
      <div className="insight-card">
        <h3>⚠️ High Risk Users</h3>
        {insights.highRiskUsers.map(user => (
          <div key={user.userId} className="risk-item">
            <span>{user.name}</span>
            <span className="risk-badge">{user.riskLevel}</span>
            <small>{user.reasons.join(', ')}</small>
          </div>
        ))}
      </div>
      
      <div className="insight-card">
        <h3>🔍 Anomaly Detections</h3>
        {insights.anomalies.map((anomaly, idx) => (
          <div key={idx} className="anomaly-item">
            <span>{anomaly.userId}</span>
            <span>Score: {anomaly.score.toFixed(2)}</span>
            <p>{anomaly.description}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AIRiskDashboard;
```

## Backend Database Schema Examples

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
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  loginAttempts: { type: Number, default: 0 },
  lastLogin: { type: Date },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  dueDate: { type: Date },
  timeTracked: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  raisedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general'], 
    default: 'general' 
  },
  resolution: { type: String },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: { type: Date }
});

module.exports = mongoose.model('Ticket', ticketSchema);
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
                ('classifier', MultinomialNB())
            ])
    
    def train(self, texts, labels):
        self.pipeline.fit(texts, labels)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.pipeline, f)
    
    def predict(self, text):
        category = self.pipeline.predict([text])[0]
        probabilities = self.pipeline.predict_proba([text])[0]
        confidence = max(probabilities)
        return {
            'category': category,
            'confidence': float(confidence)
        }
```

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier

class RiskPredictor:
    def __init__(self):
        self.model = RandomForestClassifier(n_estimators=100, random_state=42)
        self.is_trained = False
    
    def extract_features(self, user_data):
        features = [
            user_data.get('loginAttempts', 0),
            user_data.get('tasksCompleted', 0),
            user_data.get('ticketsRaised', 0),
            user_data.get('averageTaskTime', 0),
            len(user_data.get('accessPatterns', [])),
            1 if self.is_unusual_time(user_data.get('lastLoginTime')) else 0
        ]
        return np.array(features).reshape(1, -1)
    
    def is_unusual_time(self, login_time):
        if not login_time:
            return False
        hour = int(login_time.split('T')[1].split(':')[0])
        return hour < 6 or hour > 22
    
    def predict(self, user_data):
        if not self.is_trained:
            return {'riskScore': 0.5, 'riskLevel': 'medium'}
        
        features = self.extract_features(user_data)
        risk_score = float(self.model.predict_proba(features)[0][1])
        
        if risk_score < 0.3:
            risk_level = 'low'
        elif risk_score < 0.7:
            risk_level = 'medium'
        else:
            risk_level = 'high'
        
        return {
            'riskScore': risk_score,
            'riskLevel': risk_level,
            'factors': self.get_risk_factors(user_data, risk_score)
        }
    
    def get_risk_factors(self, user_data, score):
        factors = []
        if user_data.get('loginAttempts', 0) > 3:
            factors.append('Multiple failed login attempts')
        if self.is_unusual_time(user_data.get('lastLoginTime')):
            factors.append('Login at unusual time')
        if user_data.get('ticketsRaised', 0) > 5:
            factors.append('High number of support tickets')
        return factors
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
class BurnoutDetector:
    def analyze(self, user_data):
        score = 0
        reasons = []
        
        # Check weekly hours
        weekly_hours = user_data.get('weeklyHours', 40)
        if weekly_hours > 50:
            score += 2
            reasons.append(f'Excessive work hours: {weekly_hours}/week')
        
        # Check task overload
        tasks_assigned = user_data.get('tasksAssigned', 0)
        tasks_completed = user_data.get('tasksCompleted', 0)
        completion_rate = tasks_completed / tasks_assigned if tasks_assigned > 0 else 1
        
        if completion_rate < 0.6:
            score += 2
            reasons.append(f'Low task completion rate: {completion_rate:.0%}')
        
        # Check overdue tasks
        overdue_count = user_data.get('overdueTasksCount', 0)
        if overdue_count > 5:
            score += 1
            reasons.append(f'{overdue_count} overdue tasks')
        
        # Check work-life balance
        if user_data.get('weekendWork', False):
            score += 1
            reasons.append('Regular weekend work detected')
        
        night_hours = user_data.get('nightWorkHours', 0)
        if night_hours > 10:
            score += 1
            reasons.append(f'{night_hours} hours of night work')
        
        # Determine risk level
        if score <= 2:
            risk = 'low'
        elif score <= 4:
            risk = 'moderate'
        elif score <= 6:
            risk = 'high'
        else:
            risk = 'critical'
        
        recommendations = self.get_recommendations(risk, reasons)
        
        return {
            'burnoutRisk': risk,
            'score': score,
            'reasons': reasons,
            'recommendations': recommendations
        }
    
    def get_recommendations(self, risk, reasons):
        if risk in ['high', 'critical']:
            return [
                'Redistribute workload immediately',
                'Schedule 1-on-1 with manager',
                'Consider reducing task assignments',
                'Encourage time off or break'
            ]
        elif risk == 'moderate':
            return [
                'Monitor workload closely',
                'Check in on work-life balance',
                'Review task priorities'
            ]
        return ['Continue regular check-ins']
```

## Configuration

### Environment Variables

**Backend (.env)**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your_strong_secret_key_min_32_chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```env
MODEL_PATH=./models
LOG_LEVEL=info
BACKEND_URL=http://localhost:5000
MAX_WORKERS=4
CACHE_TTL=3600
```

**Frontend (.env)**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Common Troubleshooting

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
    console.error('MongoDB connection error:', error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Authentication Middleware
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);
    
    if (!user || user.status !== 'active') {
      return res.status(401).json({ error: 'Invalid authentication' });
    }
    
    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, requireAdmin };
```

### CORS Configuration
```javascript
// backend/app.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));

app.use(express.json());
```

### ML Service Health Check
```python
# ml-service/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "ml-analytics",
        "version": "1.0.0"
    }
```

### Fix: Model Not Loading
```python
# Ensure models directory exists
import os

def ensure_model_directory():
    model_path = os.getenv('MODEL_PATH', './models')
    if not os.path.exists(model_path):
        os.makedirs(model_path)
        print(
