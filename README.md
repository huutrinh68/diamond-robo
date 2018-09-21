# diamond-robo-gunn
```javascript
public void run() {
    try {
      ErrorLogger.init(this);
      initComponents();
      initColors();

      # ロボット回転する時に、大砲が逆方向に回転
      # 大砲が回転する時にRadarが逆方向に回転
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
