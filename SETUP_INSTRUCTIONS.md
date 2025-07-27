# Out of Box - Setup Instructions

## 🚀 Ready to Start Building!

Your project documentation is complete and committed to Git. Now let's create the actual iOS app.

## Step 1: Create Xcode Project (5 minutes)

### Open Xcode and Create Project:
1. **Open Xcode** (should be installed)
2. Choose **"Create a new Xcode project"**
3. Select **"iOS"** → **"App"**
4. Fill in project details:
   - **Product Name**: `OutOfBox`
   - **Team**: Select your Apple Developer team (or personal team)
   - **Organization Identifier**: `com.jiawenzhan.outofbox`
   - **Bundle Identifier**: `com.jiawenzhan.outofbox` (auto-generated)
   - **Language**: `Swift`
   - **Interface**: `SwiftUI`
   - **Use Core Data**: ❌ **NO** (we'll use SwiftData)
5. **Choose Save Location**: 
   - Navigate to: `/Users/jiawenzhan/Desktop/out_of_box_project/ios-app/`
   - Click **"Create"**

### Test the Project:
1. In Xcode, press **Cmd+R** (or click ▶️)
2. Choose **iPhone 15 Pro** simulator
3. **Success!** You should see "Hello, world!" in the simulator

## Step 2: Connect to GitHub (10 minutes)

### Option A: GitHub Desktop (Easiest)
1. **Download GitHub Desktop**: https://desktop.github.com/
2. **Install and sign in** with your GitHub account
3. **Add existing repository**: 
   - Click "Add" → "Add Existing Repository"
   - Choose: `/Users/jiawenzhan/Desktop/out_of_box_project/`
4. **Publish to GitHub**:
   - Click "Publish repository"
   - Name: `out-of-box-app`
   - Description: `Daily challenge iOS app with AI personalization`
   - Keep it **public** or **private** (your choice)
   - Click "Publish Repository"

### Option B: Terminal/Command Line
```bash
# Navigate to project
cd /Users/jiawenzhan/Desktop/out_of_box_project

# Create GitHub repository (you'll need to do this on github.com first)
# Go to github.com → New Repository → Name: "out-of-box-app"

# Add remote origin (replace YOUR_USERNAME with your GitHub username)
git remote add origin https://github.com/YOUR_USERNAME/out-of-box-app.git

# Push to GitHub
git branch -M main
git push -u origin main
```

### Option C: Xcode Built-in Git
1. In Xcode: **Source Control** → **New Git Repositories**
2. Choose your project
3. **Source Control** → **New Remote**
4. Add your GitHub repository URL

## Step 3: Setup VS Code Integration (5 minutes)

### Install VS Code Extensions:
1. **Open VS Code**
2. **Press Cmd+Shift+X** (Extensions panel)
3. **Search and install**:
   - `Swift` by Swift Server Work Group
   - `GitLens` by GitKraken
   - `Bracket Pair Colorizer 2` by CoenraadS (optional)

### Open Project in VS Code:
1. **File** → **Open Folder**
2. Navigate to: `/Users/jiawenzhan/Desktop/out_of_box_project/`
3. Click **"Open"**

### Test Hybrid Workflow:
1. **In VS Code**: Open `ios-app/OutOfBox/ContentView.swift`
2. **Change text** from "Hello, world!" to "Hello, Out of Box!"
3. **Save** (Cmd+S)
4. **Switch to Xcode** → **Press Cmd+R**
5. **Success!** Simulator should show your change

## Step 4: Project Structure Setup (Optional)

### Organize Your Workspace:
```
out_of_box_project/
├── 📁 ios-app/           # Xcode project goes here
│   └── OutOfBox.xcodeproj
├── 📁 documentation/     # Move docs here (optional)
├── 📄 README.md
├── 📄 TODO.md
└── 📄 QUICK_START.md
```

## What You Should Have Now:

✅ **Git repository** with all documentation  
✅ **GitHub repository** (public or private)  
✅ **Xcode project** that builds and runs  
✅ **VS Code** set up for editing  
✅ **Hybrid workflow** tested and working  

## Next Steps:

1. **Follow QUICK_START.md** to build your first features
2. **Use TODO.md** to track development progress  
3. **Commit regularly** as you build features
4. **Test in simulator** after every change

## Troubleshooting:

### Xcode Issues:
- **"Development team required"**: Add your Apple ID in Xcode preferences
- **"Bundle identifier already exists"**: Change to `com.jiawenzhan.outofbox.unique`
- **Simulator won't launch**: Try restarting Xcode

### VS Code Issues:
- **Swift syntax not working**: Install Swift extension and restart VS Code
- **Git not showing changes**: Make sure you opened the correct folder
- **File changes not detected in Xcode**: Save in VS Code first

### GitHub Issues:
- **Authentication failed**: Use GitHub Desktop or personal access token
- **Repository already exists**: Choose a different name or delete existing repo

## 🎉 You're Ready to Build!

Your development environment is now set up. Time to start building Out of Box! 

**Next:** Open `QUICK_START.md` and start with "Phase 2: Add Real Functionality"