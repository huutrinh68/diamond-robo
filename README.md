# diamond-robo-gunn
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
###### _gun.execute()

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
