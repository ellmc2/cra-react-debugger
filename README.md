# cra-react-debugger

基于 cra 脚手架创建 react 项目并调试react@18.2.0源码

## 一、准备

```sh
# 1、初始化项目
npx create-react-app cra-debugger

# 2、使用cra提供的自定义配置
npm run eject

# 3、拉取v18.2.0 tag 的react
cd src && git clone https://github.com/facebook/react.git 
# 更新分支并切换并创建18.2.0分支
git remote update && git checkout -b v18.2.0 v18.2.0
# 删除react文件夹下的.git文件夹

# 注意：需要切换node版本至v16，否则会报错：Current node version is not supported for development, expected "18.14.0" to satisfy "^12.17.0 || 13.x || 14.x || 15.x || 16.x || 17.x".
nvm use v16


# 4、打包react、scheduler、react-dom成cjs包
yarn build react/index,react/jsx,react-dom/index,scheduler --type=NODE

# 5、在源码目录 build/node_modules 为 react、react-dom 创建 yarn link
# 通过yarn link 可以改变项目中依赖包的目录指向
# 声明react指向
cd build/node_modules/react && yarn link
# 声明react-dom指向
cd build/node_modules/react-dom && yarn link

# 6、进入项目根目录，通过yarn link 将项目中的react、react-dom的指向刚刚打包好的react、react-dom
yarn link react react-dom

# 7、修改src/react/build/node_modules/react/cjs/react.development.js添加console.log，启动项目发现执行了打印

```

