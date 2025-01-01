
```
# Change to your Override directory
cd $HOME\Desktop\HHC2024\hhc24-snowballshowdown.holidayhackchallenge.com\js\

# Create the diff file
@"
--- .\phaser-snowball-game.js
+++ .\phaser-snowball-game.js
@@ -14,1 +14,1 @@
-        this.playerMoveSpeed = 150;
+        this.playerMoveSpeed = 450;

@@ -17,1 +17,1 @@
-        this.percentageShotPower = 0;
+        this.percentageShotPower = 100;

@@ -20,1 +20,1 @@
-        this.wombleyElvesThrowDelayMin = 1500;
+        this.wombleyElvesThrowDelayMin = 150000;

@@ -21,1 +21,1 @@
-        this.wombleyElvesThrowDelayMax = 2500;
+        this.wombleyElvesThrowDelayMax = 250000;

@@ -25,1 +25,1 @@
-        this.throwSpeed = 1000;
+        this.throwSpeed = 4000;

@@ -26,1 +26,1 @@
-        this.throwRateOfFire = 1000;
+        this.throwRateOfFire = 10;

@@ -112,1 +112,1 @@
-        let numOfWombleyElves = 10;
+        let numOfWombleyElves = 2;

@@ -679,1 +679,1 @@
-        this.snowBallBlastRadius = 24;
+        this.snowBallBlastRadius = 500;

@@ -680,1 +680,1 @@
-        this.onlyMoveHorizontally = true;
+        this.onlyMoveHorizontally = false;
"@ | Out-File -FilePath "phaser-snowball-game.js.diff" -Encoding UTF8

# Apply the diff. If this step fails, ensure that you have loaded the windiff function shared earlier
windiff -apply phaser-snowball-game.js .\phaser-snowball-game.js.diff

```