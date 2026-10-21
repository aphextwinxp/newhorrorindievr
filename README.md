Сюда кладётся Three.js r160 (`three.module.js`). Выполните `npm run vendor` в корне проекта
и закоммитьте результат — GitHub Pages не умеет подтягивать пакеты из npm.
Папка `addons/` нужна только если в коде появятся импорты `three/addons/...` (`npm run vendor:addons`).
