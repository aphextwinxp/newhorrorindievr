# Ночная смена — Эпизод 1: Последний клиент

WebXR-хоррор для Meta Quest и ПК на Three.js r160. Без сборщика: нативные ES-модули и importmap.

## Запуск

```bash
npm run vendor     # один раз: кладёт three.module.js в libs/three/ (закоммитьте результат)
npm start          # http://localhost:3000
npm test           # проверки логики уровня и XR (без браузера)
```

WebXR работает только в безопасном контексте: `localhost` или HTTPS. Варианты для Quest:

- GitHub Pages (HTTPS из коробки) — самый простой путь, включите Pages для ветки в настройках репозитория;
- USB + `adb reverse tcp:3000 tcp:3000`, затем открыть `http://localhost:3000` в браузере шлема.

Отладка: добавьте `?debug=1` к адресу — появится оверлей с FPS и draw calls, а в консоли `window.NightShift`.

## Структура

```
index.html            страница, importmap, сторож загрузки
css/style.css         стили HTML-интерфейса (меню, HUD ПК)
libs/three/           Three.js r160 (vendored)
src/
  config.js           все константы и схема настроек
  game.js             ядро: рендерер, цикл, реестр систем, WebXR-сессия
  ui.js               HTML-интерфейс
  main.js             точка входа, регистрация систем
  xr.js               VR-контроллеры, ввод, вибрация, поза головы        (новое)
  vr-hud.js           3D-часы на запястье и субтитры в VR                 (новое)
  level-data.js       данные АЗС: комнаты, стены, двери, лампы, точки     (новое)
  level.js            постройка уровня, коллизии, свет, питание, чекпоинты (новое)
tests/                headless-проверки на заглушке Three.js
```

## Системы и порядок вызова

| Система | Приоритет | Назначение |
|---|---|---|
| `xr` | -100 | опрос контроллеров первым, чтобы все видели свежий ввод |
| `level` | -50 | свет, питание, чекпоинты |
| `vrHud` | 100 | панели VR, `lateUpdate` после всех |

Позже: `input.js` / `player.js` (около 0), `interact.js`, `nav.js`, `enemy.js`, `audio.js`, `story.js`.

## API для следующих модулей

**`game.getSystem('xr')`**

- `getStick('left'|'right')` → `{x, y}`, `y > 0` — вперёд, мёртвая зона уже учтена
- `isDown / justPressed / justReleased / getValue(hand, 'trigger'|'squeeze'|'thumbstick'|'primary'|'secondary')`
  — «краевые» состояния живут один кадр `update`, читать их нужно в `update`, а не в `fixedUpdate`
- `getHeadWorldPosition(v)`, `getHeadForward(v)`, `getHeadYaw()`, `getHeadHeight()` — поза из XRFrame, без задержки кадра
- `getHandWorldPosition(hand, v)`, `getHandForward(hand, v)`, `getGrip(hand)`, `getRay(hand)`
- `pulse(hand, intensity, ms)`, `pulsePreset(hand, 'grab'|'doorSlam')`
- события: `xrbutton`, `xrcontroller`

**`game.getSystem('level')`**

- коллизии: `resolveCircle({x, z}, radius)`, `segmentBlocked(x0, z0, x1, z1)` (видимость врага), `getColliders()`
- двери: `getDoor(id)`, `setDoorAngle(id, rad)`, `setDoorLocked(id, bool)`, `isDoorClosed(id)`
- питание и свет: `setPower(bool)`, `blackout(seconds)`, `forceLamp(id, 'on'|'off'|null)`, `forceRoomLamps(room, state)`, `setFlickerOverride(0..1|null)`
- данные: `getRoomAt(x, z)`, `getInteractables()`, `getHidingSpots()`, `getPatrolPoints()`, `getEnemySpawns()`, `getNavBounds()`
- события: `powerchange`, `checkpointreached`

**Субтитры:** `game.emit('subtitle', { text, seconds })` — `main.js` покажет их в HTML (ПК) или на 3D-панели (VR).

## Как устроен свет

Бюджет динамических источников у Quest 2 — 3 (один занят фонариком), поэтому ламп много, а реальных `PointLight` мало:
пул источников «переезжает» к ближайшим включённым лампам и плавно гаснет перед переездом. Остальные лампы светятся эмиссией.
Мерцание зависит от акта (`actFlicker` в `level-data.js`). Статика (стены, полы, мебель) собрана в 10 мешей — по одному на материал.

## Что нужно проверить на шлеме

Всё ниже — стартовые значения, их нельзя было проверить без устройства:

- ориентация и размер часов на запястье (`WATCH` в `vr-hud.js`), расстояние субтитров (`SUBTITLE`);
- размеры проёмов, высота прилавка и мебели в `level-data.js`;
- яркость ламп и дальность света (`CONFIG.ATMOSPHERE`);
- draw calls по оверлею `?debug=1` (по расчёту около 35 на глаз, бюджет Quest 2 — 100).

## Известные ограничения

- Кнопка меню левого контроллера на Quest зарезервирована системой и, насколько я знаю, WebXR её не отдаёт.
  В таблице управления в `index.html` она указана для паузы в VR; стоит выбрать другой жест или убрать строку.
- `game.restartFromCheckpoint()` ставит rig в точку чекпоинта, а в VR голова может стоять со смещением (room-scale):
  `player.js` должен это компенсировать.
- Флаг адаптации `cullFarRooms` выставляется, но уровень пока не скрывает дальние комнаты.
- Чекпоинт пишется в `localStorage`, но кнопки «Продолжить» в меню нет (`level.loadSavedCheckpoint()` готов).
- Предметы-заглушки под сюжет (`office_key`, `evidence_1…4`, `terminal`, `phone`, `car_door`) — их назначение определит `story.js`.
