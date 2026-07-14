---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and predictive insights
triggers:
  - "setup enterprise user management system"
  - "implement AI-powered task management dashboard"
  - "create user management system with analytics"
  - "build admin dashboard with role-based access"
  - "integrate AI ticket classification and routing"
  - "setup kanban board with time tracking"
  - "implement burnout detection and risk prediction"
  - "configure JWT authentication for user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for enterprise user, task, and support ticket management with integrated AI analytics for risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, secure authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring

**Stack**: React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance

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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
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
DATABASE_URL=${MONGODB_URI}
MODEL_PATH=./models
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

ML service runs at `http://localhost:8000`

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

Frontend runs at `http://localhost:3000`

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }

// Get current user
GET /api/auth/me
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Get user by ID
GET /api/users/:id
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Update user
PUT /api/users/:id
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }
{
  "username": "jane.doe",
  "email": "jane@example.com",
  "role": "admin"
}

// Delete user
DELETE /api/users/:id
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }
{
  "title": "Implement authentication",
  "description": "Add JWT authentication to API",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo" // todo, in-progress, done
}

// Get tasks
GET /api/tasks
GET /api/tasks?status=in-progress
GET /api/tasks?assignedTo=user_id

// Update task
PUT /api/tasks/:id
{
  "status": "in-progress",
  "timeSpent": 120 // minutes
}

// Delete task
DELETE /api/tasks/:id
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }
{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical" // technical, billing, general
}

// Get tickets
GET /api/tickets
GET /api/tickets?status=open
GET /api/tickets?userId=user_id

// Update ticket
PUT /api/tickets/:id
{
  "status": "in-progress",
  "assignedTo": "admin_id",
  "response": "Working on this issue"
}
```

## ML Service API Reference

### AI Ticket Classification

```python
# Classify ticket automatically
POST /api/ml/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin dashboard"
}
# Returns: { category: "technical", priority: "high", confidence: 0.85 }
```

### Risk Prediction

```python
# Predict user risk score
POST /api/ml/risk-prediction
{
  "userId": "user_id",
  "failedLogins": 3,
  "suspiciousActivities": 1,
  "lastActiveHours": 72
}
# Returns: { riskScore: 0.75, riskLevel: "high", factors: [...] }
```

### Anomaly Detection

```python
# Detect anomalous behavior
POST /api/ml/anomaly-detection
{
  "userId": "user_id",
  "activityData": {
    "loginTime": "2026-04-15T03:00:00Z",
    "ipAddress": "192.168.1.100",
    "location": "Unknown",
    "tasksCompleted": 50
  }
}
# Returns: { isAnomaly: true, anomalyScore: 0.92, reason: "Unusual login time" }
```

### Burnout Detection

```python
# Analyze burnout risk
POST /api/ml/burnout-detection
{
  "userId": "user_id",
  "workloadData": {
    "tasksAssigned": 25,
    "tasksCompleted": 18,
    "averageTaskTime": 240,
    "overtimeHours": 15,
    "weekendWork": true
  }
}
# Returns: { burnoutRisk: "high", score: 0.82, recommendations: [...] }
```

### Project Delay Prediction

```python
# Predict project delays
POST /api/ml/predict-delays
{
  "projectId": "project_id",
  "totalTasks": 50,
  "completedTasks": 20,
  "daysElapsed": 30,
  "totalDays": 60
}
# Returns: { delayProbability: 0.65, estimatedDelay: 15, factors: [...] }
```

## Frontend Components

### Authentication Component

```javascript
// src/components/Login.js
import React, { useState } from 'react';
import axios from 'axios';

const Login = () => {
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  
  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/api/auth/login`,
        credentials
      );
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      window.location.href = '/dashboard';
    } catch (error) {
      console.error('Login failed:', error.response?.data?.message);
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <input
        type="email"
        placeholder="Email"
        value={credentials.email}
        onChange={(e) => setCredentials({...credentials, email: e.target.value})}
      />
      <input
        type="password"
        placeholder="Password"
        value={credentials.password}
        onChange={(e) => setCredentials({...credentials, password: e.target.value})}
      />
      <button type="submit">Login</button>
    </form>
  );
};

export default Login;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const organized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(organized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
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

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({});
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [risk, burnout, anomalies] = await Promise.all([
        axios.post(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`,
          { userId },
          { headers: { Authorization: `Bearer ${token}` } }
        ),
        axios.post(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`,
          { userId, workloadData: {} },
          { headers: { Authorization: `Bearer ${token}` } }
        ),
        axios.get(
          `${process.env.REACT_APP_API_URL}/api/analytics/anomalies/${userId}`,
          { headers: { Authorization: `Bearer ${token}` } }
        )
      ]);

      setAnalytics({
        risk: risk.data,
        burnout: burnout.data,
        anomalies: anomalies.data
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Level</h3>
        <p className={`risk-${analytics.risk?.riskLevel}`}>
          {analytics.risk?.riskLevel?.toUpperCase()}
        </p>
        <span>Score: {analytics.risk?.riskScore?.toFixed(2)}</span>
      </div>
      
      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <p className={`burnout-${analytics.burnout?.burnoutRisk}`}>
          {analytics.burnout?.burnoutRisk?.toUpperCase()}
        </p>
        <span>Score: {analytics.burnout?.score?.toFixed(2)}</span>
      </div>
      
      <div className="metric-card">
        <h3>Anomalies Detected</h3>
        <p>{analytics.anomalies?.count || 0}</p>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No token provided');
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user) {
      throw new Error('User not found');
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Please authenticate' });
  }
};

const adminAuth = async (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getTasks = async (req, res) => {
  try {
    const { status, assignedTo, priority } = req.query;
    const filter = {};
    
    if (status) filter.status = status;
    if (assignedTo) filter.assignedTo = assignedTo;
    if (priority) filter.priority = priority;
    
    // Non-admins can only see their tasks
    if (req.user.role !== 'admin') {
      filter.assignedTo = req.user.id;
    }

    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Check permissions
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    Object.assign(task, req.body);
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
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
    def __init__(self):
        self.model = None
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            with open(f'{model_path}/ticket_classifier.pkl', 'rb') as f:
                self.model = pickle.load(f)
        except FileNotFoundError:
            self.train_initial_model()
    
    def train_initial_model(self):
        # Initialize with basic model
        self.model = Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000)),
            ('clf', MultinomialNB())
        ])
    
    def classify(self, title: str, description: str) -> dict:
        text = f"{title} {description}"
        
        if not hasattr(self.model, 'predict'):
            # Default classification for untrained model
            return {
                'category': 'general',
                'priority': 'medium',
                'confidence': 0.5
            }
        
        category = self.model.predict([text])[0]
        confidence = max(self.model.predict_proba([text])[0])
        
        # Determine priority based on keywords
        priority = self._determine_priority(text)
        
        return {
            'category': category,
            'priority': priority,
            'confidence': float(confidence)
        }
    
    def _determine_priority(self, text: str) -> str:
        text_lower = text.lower()
        
        high_priority_keywords = ['urgent', 'critical', 'down', 'error', 'cannot']
        medium_priority_keywords = ['issue', 'problem', 'slow']
        
        if any(kw in text_lower for kw in high_priority_keywords):
            return 'high'
        elif any(kw in text_lower for kw in medium_priority_keywords):
            return 'medium'
        return 'low'
```

### FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import Dict, List
from models.ticket_classifier import TicketClassifier
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector

app = FastAPI(title="ML Analytics Service")

# Initialize models
ticket_classifier = TicketClassifier()
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()

class TicketData(BaseModel):
    title: str
    description: str

class RiskData(BaseModel):
    userId: str
    failedLogins: int = 0
    suspiciousActivities: int = 0
    lastActiveHours: float = 0

class BurnoutData(BaseModel):
    userId: str
    workloadData: Dict

@app.post("/api/ml/classify-ticket")
async def classify_ticket(data: TicketData):
    try:
        result = ticket_classifier.classify(data.title, data.description)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/risk-prediction")
async def predict_risk(data: RiskData):
    try:
        features = {
            'failed_logins': data.failedLogins,
            'suspicious_activities': data.suspiciousActivities,
            'last_active_hours': data.lastActiveHours
        }
        result = risk_predictor.predict(data.userId, features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(data: BurnoutData):
    try:
        result = burnout_detector.analyze(data.userId, data.workloadData)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
NODE_ENV=production
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

### ML Service Environment Variables

```bash
# ml-service/.env
DATABASE_URL=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
RETRAIN_INTERVAL=86400
```

## Common Patterns

### API Request Helper

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const getAuthHeaders = () => ({
  Authorization: `Bearer ${localStorage.getItem('token')}`
});

export const api = {
  // Auth
  login: (credentials) => 
    axios.post(`${API_URL}/api/auth/login`, credentials),
  
  register: (userData) => 
    axios.post(`${API_URL}/api/auth/register`, userData),
  
  // Tasks
  getTasks: (filters = {}) =>
    axios.get(`${API_URL}/api/tasks`, {
      headers: getAuthHeaders(),
      params: filters
    }),
  
  createTask: (taskData) =>
    axios.post(`${API_URL}/api/tasks`, taskData, {
      headers: getAuthHeaders()
    }),
  
  updateTask: (taskId, updates) =>
    axios.put(`${API_URL}/api/tasks/${taskId}`, updates, {
      headers: getAuthHeaders()
    }),
  
  // Tickets
  createTicket: (ticketData) =>
    axios.post(`${API_URL}/api/tickets`, ticketData, {
      headers: getAuthHeaders()
    }),
  
  // ML Analytics
  classifyTicket: (title, description) =>
    axios.post(`${ML_API_URL}/api/ml/classify-ticket`, {
      title,
      description
    }),
  
  getRiskPrediction: (userId, data) =>
    axios.post(`${ML_API_URL}/api/ml/risk-prediction`, {
      userId,
      ...data
    }, {
      headers: getAuthHeaders()
    })
};
```

### Custom Hooks

```javascript
// frontend/src/hooks/useTasks.js
import { useState, useEffect } from 'react';
import { api } from '../utils/api';

export const useTasks = (filters = {}) => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchTasks = async () => {
    try {
      setLoading(true);
      const response = await api.getTasks(filters);
      setTasks(response.data);
      setError(null);
    } catch (err) {
      setError(err.response?.data?.message || 'Failed to fetch tasks');
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchTasks();
  }, [JSON.stringify(filters)]);

  return { tasks, loading, error, refetch: fetchTasks };
};
```

## Troubleshooting

### Connection Issues

**Problem**: Frontend cannot connect to backend
```bash
# Check backend is running
curl http://localhost:5000/health

# Check CORS configuration
# backend/server.js
const cors = require('cors');
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000'
}));
```

**Problem**: ML service not responding
```bash
# Check ML service health
curl http://localhost:8000/health

# Check logs
cd ml-service
uvicorn main:app --reload --log-level debug
```

### Authentication Errors

**Problem**: Token expired or invalid
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

### Database Connection

**Problem**: Cannot connect to MongoDB
```bash
# Check MongoDB connection string format
# mongodb://username:password@host:port/database
# mongodb+srv://username:password@cluster.mongodb.net/database

# Test connection
node -e "require('mongoose').connect(process.env.MONGODB_URI).then(() => console.log('Connected')).catch(err => console.error(err))"
```

### Model Training Issues

**Problem**: ML models not loading
```bash
# Ensure models directory exists
mkdir -p ml-service/models

# Check model files
ls -la ml-service/models/

# Retrain models
cd ml-service
python train_models.py
```

### Performance Optimization

**Slow API responses**:
```javascript
// Add pagination to task queries
GET /api/tasks?page=1&limit=20

// Backend implementation
exports.getTasks = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;

  const tasks = await Task.find(filter)
    .skip(skip)
    .limit(limit)
    .sort({ createdAt: -1 });
  
  const total = await Task.countDocuments(filter);
  
  res.json({
    tasks,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
};
```

## Deployment

### Docker Deployment

```dockerfile
# backend/Dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=${MONGODB_URI}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - mongo
  
  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=${MONGODB_URI}
  
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://backend:5000
  
  mongo:
    image: mongo:5
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

Run with Docker:
```bash
docker-compose up -d
```
