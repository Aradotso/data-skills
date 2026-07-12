---
name: enterprise-user-management-ai-system
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "configure user dashboard with task tracking"
  - "implement JWT authentication for user system"
  - "add AI-based ticket classification"
  - "create admin dashboard with analytics"
  - "detect user burnout with machine learning"
  - "build kanban board for task management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in deploying and customizing a full-stack enterprise user management system with integrated AI analytics. The system manages users, tasks, and support tickets while providing intelligent insights through machine learning models for risk detection, anomaly detection, burnout analysis, and predictive project analytics.

## What This Project Does

The Enterprise User Management System is a three-tier application consisting of:

- **Frontend (React)**: User and admin dashboards with Kanban boards, time tracking, and analytics visualization
- **Backend (Node.js)**: REST API for user management, authentication (JWT), task assignment, and ticket handling
- **ML Service (FastAPI)**: AI-powered analytics including ticket classification, risk prediction, anomaly detection, and burnout analysis

Key capabilities:
- Secure user authentication and role-based access control
- Task management with drag-and-drop Kanban interface
- Support ticket system with AI-based routing
- Real-time analytics and performance insights
- Predictive models for project delays and user burnout
- Audit logging and security monitoring

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance (local or cloud)

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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start the backend:

```bash
npm start
# or for development with auto-reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start the ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start the frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const response = await fetch(`${API_URL}/api/auth/register`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: "John Doe",
    email: "john@example.com",
    password: "SecurePass123",
    role: "user" // or "admin"
  })
});
```

**Login**
```javascript
// POST /api/auth/login
const response = await fetch(`${API_URL}/api/auth/login`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: "john@example.com",
    password: "SecurePass123"
  })
});
const { token, user } = await response.json();
// Store token for authenticated requests
localStorage.setItem('token', token);
```

### User Management (Admin)

**Get All Users**
```javascript
// GET /api/users
const response = await fetch(`${API_URL}/api/users`, {
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  }
});
const users = await response.json();
```

**Update User**
```javascript
// PUT /api/users/:id
const response = await fetch(`${API_URL}/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: "Jane Doe",
    role: "admin",
    status: "active"
  })
});
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
const response = await fetch(`${API_URL}/api/tasks`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: "Implement login feature",
    description: "Add JWT-based authentication",
    assignedTo: userId,
    priority: "high",
    status: "todo",
    dueDate: "2026-05-01"
  })
});
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const response = await fetch(`${API_URL}/api/tasks/${taskId}/status`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: "in-progress" // or "done"
  })
});
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const response = await fetch(`${API_URL}/api/tickets`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    subject: "Cannot access reports",
    description: "Getting 404 error when clicking on analytics",
    priority: "medium",
    category: "technical"
  })
});
```

### AI Analytics Endpoints

**Classify Ticket (Auto-routing)**
```python
# POST /ml/classify-ticket
import requests

response = requests.post(
    "http://localhost:8000/ml/classify-ticket",
    json={
        "subject": "Cannot login to my account",
        "description": "Password reset not working, tried multiple times"
    }
)
classification = response.json()
# Returns: {"category": "technical", "priority": "high", "assignedDepartment": "IT Support"}
```

**Risk Detection**
```python
# POST /ml/risk-detection
response = requests.post(
    "http://localhost:8000/ml/risk-detection",
    json={
        "userId": "user123",
        "loginAttempts": 5,
        "failedLogins": 3,
        "accessPatterns": ["unusual_time", "new_location"],
        "recentActivities": ["data_export", "permission_change"]
    }
)
risk_score = response.json()
# Returns: {"riskScore": 0.78, "riskLevel": "high", "flags": ["suspicious_login", "data_access"]}
```

**Burnout Detection**
```python
# POST /ml/burnout-detection
response = requests.post(
    "http://localhost:8000/ml/burnout-detection",
    json={
        "userId": "user123",
        "hoursWorked": 65,
        "tasksCompleted": 45,
        "tasksOverdue": 12,
        "avgTaskCompletionTime": 8.5,
        "weeklyTrend": [50, 55, 60, 65, 65]
    }
)
burnout_analysis = response.json()
# Returns: {"burnoutRisk": 0.72, "recommendation": "reduce_workload", "suggestedActions": [...]}
```

**Anomaly Detection**
```python
# POST /ml/anomaly-detection
response = requests.post(
    "http://localhost:8000/ml/anomaly-detection",
    json={
        "userId": "user123",
        "timestamp": "2026-04-15T14:30:00Z",
        "activityType": "file_download",
        "dataVolume": 5000,
        "ipAddress": "192.168.1.50",
        "deviceInfo": "Unknown Device"
    }
)
anomaly = response.json()
# Returns: {"isAnomaly": true, "confidence": 0.89, "anomalyType": "unusual_activity"}
```

**Project Delay Prediction**
```python
# POST /ml/predict-delay
response = requests.post(
    "http://localhost:8000/ml/predict-delay",
    json={
        "projectId": "proj123",
        "tasksTotal": 50,
        "tasksCompleted": 20,
        "tasksInProgress": 15,
        "daysRemaining": 30,
        "teamSize": 5,
        "avgVelocity": 2.5
    }
)
prediction = response.json()
# Returns: {"delayProbability": 0.65, "estimatedDelay": 7, "recommendations": [...]}
```

## Common Patterns

### Protected Route Component (React)

```javascript
// frontend/src/components/ProtectedRoute.js
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board State Management

```javascript
// frontend/src/components/KanbanBoard.js
import { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const handleDragEnd = async (result) => {
    if (!result.destination) return;
    
    const { source, destination, draggableId } = result;
    
    if (source.droppableId !== destination.droppableId) {
      const newStatus = destination.droppableId;
      
      await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${draggableId}/status`, {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      });
      
      fetchTasks();
    }
  };

  return (
    // DragDropContext and Droppable components here
  );
};
```

### Backend Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### ML Model Training Pipeline

```python
# ml-service/models/burnout_predictor.py
from sklearn.ensemble import RandomForestClassifier
from river import linear_model, preprocessing
import pickle
import os

class BurnoutPredictor:
    def __init__(self, model_path='./models/burnout_model.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            with open(model_path, 'rb') as f:
                self.model = pickle.load(f)
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
            self.is_trained = False
    
    def predict(self, features):
        """
        features: dict with keys:
        - hours_worked: float
        - tasks_completed: int
        - tasks_overdue: int
        - avg_completion_time: float
        """
        X = [[
            features['hours_worked'],
            features['tasks_completed'],
            features['tasks_overdue'],
            features['avg_completion_time']
        ]]
        
        if not hasattr(self.model, 'classes_'):
            return {'burnoutRisk': 0.0, 'recommendation': 'insufficient_data'}
        
        proba = self.model.predict_proba(X)[0][1]
        
        recommendation = 'maintain' if proba < 0.5 else 'reduce_workload' if proba < 0.7 else 'immediate_action'
        
        return {
            'burnoutRisk': float(proba),
            'recommendation': recommendation,
            'suggestedActions': self._get_suggestions(proba)
        }
    
    def _get_suggestions(self, risk_score):
        if risk_score < 0.5:
            return ['Continue current workload', 'Monitor weekly']
        elif risk_score < 0.7:
            return ['Reduce overtime hours', 'Delegate tasks', 'Schedule breaks']
        else:
            return ['Immediate workload reduction', 'Mandatory time off', 'Team rebalancing']
    
    def save_model(self):
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.model, f)
```

### FastAPI ML Service Routes

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.burnout_predictor import BurnoutPredictor
from models.ticket_classifier import TicketClassifier

app = FastAPI()

burnout_predictor = BurnoutPredictor()
ticket_classifier = TicketClassifier()

class BurnoutRequest(BaseModel):
    userId: str
    hoursWorked: float
    tasksCompleted: int
    tasksOverdue: int
    avgTaskCompletionTime: float

class TicketRequest(BaseModel):
    subject: str
    description: str

@app.post("/ml/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    try:
        features = {
            'hours_worked': request.hoursWorked,
            'tasks_completed': request.tasksCompleted,
            'tasks_overdue': request.tasksOverdue,
            'avg_completion_time': request.avgTaskCompletionTime
        }
        result = burnout_predictor.predict(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        text = f"{request.subject} {request.description}"
        result = ticket_classifier.classify(text)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### MongoDB Connection

Ensure MongoDB is running and accessible. For cloud MongoDB Atlas:

```env
MONGODB_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/enterprise_user_mgmt?retryWrites=true&w=majority
```

### JWT Configuration

Set a strong secret key:

```bash
# Generate a random secret
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Update `backend/.env`:

```env
JWT_SECRET=<generated_secret>
JWT_EXPIRE=7d
```

### CORS Configuration

For development, configure CORS in `backend/server.js`:

```javascript
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

## Troubleshooting

### Issue: "Cannot connect to MongoDB"

**Solution**: Verify MongoDB is running and connection string is correct.

```bash
# Check MongoDB status (local)
sudo systemctl status mongod

# Test connection
mongo --eval "db.adminCommand('ping')"
```

### Issue: "JWT token expired"

**Solution**: Implement token refresh mechanism:

```javascript
// frontend/src/utils/api.js
const refreshToken = async () => {
  const response = await fetch(`${API_URL}/api/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      refreshToken: localStorage.getItem('refreshToken')
    })
  });
  const { token } = await response.json();
  localStorage.setItem('token', token);
  return token;
};
```

### Issue: "ML service not responding"

**Solution**: Check if Python dependencies are installed and service is running:

```bash
cd ml-service
pip list | grep -E "fastapi|scikit-learn|river"

# Check if service is running
curl http://localhost:8000/health
```

### Issue: "Tasks not updating in real-time"

**Solution**: Implement WebSocket connection for real-time updates:

```javascript
// backend/server.js
const socketio = require('socket.io');
const io = socketio(server, {
  cors: { origin: process.env.FRONTEND_URL }
});

io.on('connection', (socket) => {
  socket.on('taskUpdate', (data) => {
    io.emit('taskUpdated', data);
  });
});
```

### Issue: "High burnout false positives"

**Solution**: Retrain model with more labeled data:

```python
# ml-service/scripts/train_burnout_model.py
from models.burnout_predictor import BurnoutPredictor
import pandas as pd

# Load training data
data = pd.read_csv('training_data/burnout_labeled.csv')
X = data[['hours_worked', 'tasks_completed', 'tasks_overdue', 'avg_completion_time']]
y = data['burnout_label']

predictor = BurnoutPredictor()
predictor.model.fit(X, y)
predictor.save_model()
print("Model retrained and saved")
```

## Deployment Considerations

### Environment Variables for Production

```env
# backend/.env
NODE_ENV=production
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=https://ml-service.yourdomain.com
FRONTEND_URL=https://app.yourdomain.com

# ml-service/.env
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=/app/models
LOG_LEVEL=WARNING
```

### Docker Deployment

Create `docker-compose.yml`:

```yaml
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
      - ml-service

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URI=${MONGODB_URI}
    volumes:
      - ./models:/app/models

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://backend:5000/api
```

This comprehensive skill enables AI agents to assist developers in setting up, configuring, and extending the Enterprise User Management System with AI Analytics.
