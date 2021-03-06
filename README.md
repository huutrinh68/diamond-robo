# [diamond-robo-gun](http://robowiki.net/wiki/Diamond)
###### using knn to predict guessFactor, then calcualate firingAngle (in PerceptualGun class, TripHammerKnnGun class)
###### _gun.execute()
1vs1: VituralGun and Melee: Melee Gun 
```javascript
  private static DiamondFist _gun;

  public void run() {
    try {
      ErrorLogger.init(this);
      initComponents();
      initColors();

      setAdjustGunForRobotTurn(true);
      setAdjustRadarForGunTurn(true);

      while (true) {
        _gfx.updatePaintProcessing();
        if (!_TC) {
          _move.execute();
        }
        if (!_MC) {
          _gun.execute();
        }
        _radar.execute();
        execute();
      }
    } catch (RuntimeException re) {
      logAndRethrowException(re);
    }
  }
```
```javascript
  public void execute() {
    _gunDataManager.execute(_robot.getRoundNum(), _robot.getTime(),
        calculateBulletPower(), _robot.getGunHeat(), myLocation(), is1v1(),
        paintStatus());
    if (is1v1()) {
      GunEnemy duelEnemy = _gunDataManager.duelEnemy();
      if (duelEnemy != null) {
        aimAndFire(duelEnemy);
        if (!_startedDuel) {
          _startedDuel = true;
          printCurrentGun(duelEnemy);
        }
      }
    } else {
      aimAndFireAtEveryone();
    }
    if (paintStatus() && _drawVictory) {
      _gunDataManager.drawVictory(_robot.getTime());
      _drawVictory = false;
    }
  }
```
###### _gunDataManager.execute()

```javascript
  public void execute(int currentRound, long currentTime, double bulletPower,
      double currentGunHeat, Point2D.Double myLocation, boolean is1v1,
      boolean painting) {
    updateBotDistances(myLocation);
    for (GunEnemy gunData : getAllEnemyData()) {
      if (gunData.alive) {
        gunData.execute(currentTime, _lastBulletFiredTime, bulletPower,
            currentGunHeat, myLocation, is1v1, _enemiesTotal, _listeners,
            painting);
      }
    }
    if (painting && _victoryGraphics.containsKey(currentTime)) {
      for (RoboGraphic graphic : _victoryGraphics.get(currentTime)) {
        _renderables.add(graphic);
      }
    }
  }
```
###### [gunData.execute](https://github.com/huutrinh68/diamond-robo/blob/master/voidious/gun/GunEnemy.java)

###### calculateBulletPower()
```javascript
  double calculateBulletPower() {
    GunEnemy gunData = _gunDataManager.getClosestLivingBot(myLocation());
    double bulletPower = 3;
    if (gunData != null) {
      double myEnergy = _robot.getEnergy();
      if (_tcMode) {
        bulletPower = Math.min(myEnergy, 3);
      } else if (is1v1()) {
        bulletPower = 1.95;

        if (gunData.distance < 150 || gunData.isRammer()) {
          bulletPower = 2.95;
        }

        if (gunData.distance > 325) {
          double powerDownPoint =
              DiaUtils.limit(35, 63 + ((gunData.energy - myEnergy) * 4), 63);
          if (myEnergy < powerDownPoint) {
            bulletPower = Math.min(bulletPower,
                DiaUtils.cube(myEnergy / powerDownPoint) * 1.95);
          }
        }

        bulletPower = Math.min(bulletPower, gunData.energy / 4);
        bulletPower = Math.max(bulletPower, 0.1);
        bulletPower = Math.min(bulletPower, myEnergy);
      } else {
        double avgEnemyEnergy = _gunDataManager.getAverageEnergy();

        bulletPower = 2.999;

        int enemiesAlive = _robot.getOthers();
        if (enemiesAlive <= 3) {
          bulletPower = 1.999;
        }

        if (enemiesAlive <= 5 && gunData.distance > 500) {
          bulletPower = 1.499;
        }

        if ((myEnergy < avgEnemyEnergy && enemiesAlive <= 5
                && gunData.distance > 300)
            || gunData.distance > 700) {
          bulletPower = 0.999;
        }

        if (myEnergy < 20 && myEnergy < avgEnemyEnergy) {
          bulletPower =
              Math.min(bulletPower, 2 - ((20 - myEnergy) / 11));
        }

        bulletPower = Math.max(bulletPower, 0.1);
        bulletPower = Math.min(bulletPower, myEnergy);
      }
    }
```
aimAndFire(), aimAndFireAtEveryone() => Find out the firingAngle, the best gun by evaluating vituralGun.
```javascript
  private void aimAndFire(GunEnemy gunData) {
    if (gunData != null) {
      fireIfGunTurned(_aimedBulletPower);

      Wave aimWave = gunData.lastWaveFired;
      _aimedBulletPower = aimWave.bulletPower();
      Point2D.Double myNextLocation = _predictor.nextLocation(_robot);
      double firingAngle;
      if (gunData.energy == 0 || ticksUntilGunCool() > 3) {
        firingAngle =
            DiaUtils.absoluteBearing(myNextLocation, aimWave.targetLocation);
        evaluateVirtualGuns(gunData);
      } else {
        firingAngle = _currentGun.aim(aimWave, paintStatus());
      }
      _robot.setTurnGunRightRadians(Utils.normalRelativeAngle(
          firingAngle - _robot.getGunHeadingRadians()));
    }
  }
```
Using meleeGun to fire the closestBot
```javascript
  private void aimAndFireAtEveryone() {
    GunEnemy closestBot = _gunDataManager.getClosestLivingBot(myLocation());
    if (closestBot != null) {
      fireIfGunTurned(_aimedBulletPower);
      Point2D.Double myNextLocation = _predictor.nextLocation(_robot);
      long ticksUntilFire = ticksUntilGunCool();

      if (ticksUntilFire % 2 == 0 || ticksUntilFire <= 4) {
        _aimedBulletPower = calculateBulletPower();
        double firingAngle = _meleeGun.aimAtEveryone(myNextLocation,
            _robot.getTime(), _robot.getOthers(), _aimedBulletPower,
            closestBot, paintStatus());
        _robot.setTurnGunRightRadians(Utils.normalRelativeAngle(
            firingAngle - _robot.getGunHeadingRadians()));
      }
    }
  }
```
###### using displacementVector in MeleeGun to find out the firingAngle
```javascript
    public double aimAtEveryone(Point2D.Double myNextLocation,
      long currentTime, int enemiesAlive, double bulletPower,
      GunEnemy closestBot, boolean painting) {
    List<MeleeFiringAngle> firingAngles = new ArrayList<MeleeFiringAngle>();

    int kSize = getCommonKsize(enemiesAlive);
    for (GunEnemy gunData : _gunDataManager.getAllEnemyData()) {
      if (gunData.alive && gunData.views.get(VIEW_NAME).size() >= 10
          && gunData.lastWaveFired != null) {
        List<MeleeFiringAngle> enemyAngles = new ArrayList<MeleeFiringAngle>();
        Wave aimWave = gunData.lastWaveFired;
        aimWave.setBulletPower(bulletPower);
        KnnView<TimestampedFiringAngle> view = gunData.views.get(VIEW_NAME);
        List<Entry<TimestampedFiringAngle>> nearestNeighbors =
            view.nearestNeighbors(aimWave, true, kSize);

        int numScans = nearestNeighbors.size();
        double totalScanWeight = 0;
        for (int x = 0; x < numScans; x++) {
          Entry<TimestampedFiringAngle> entry = nearestNeighbors.get(x);
          double scanWeight = 1 / Math.sqrt(entry.distance);
          totalScanWeight += scanWeight;
          Point2D.Double vector = entry.value.displacementVector;
          MeleeFiringAngle firingAngle = getFiringAngle(
              myNextLocation, currentTime, vector, scanWeight, aimWave);
          if (firingAngle != null) {
            enemyAngles.add(firingAngle);
          }
        }
        for (MeleeFiringAngle enemyAngle : enemyAngles) {
          enemyAngle.scanWeight /= totalScanWeight;
        }
        firingAngles.addAll(enemyAngles);
      }
   }
```