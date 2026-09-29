# Shopro商城高级版，uniapp多平台移动商城（微信公众号、微信小程序、H5网页、Android-App、IOS-App购物商城）

ym7k.com/10650/Shopro商城高级版，uniapp多平台移动商城（微信公众号、微信小程序、H5网页、Android-App、IOS-App购物商城）


<img width="964" height="898" alt="2025013015143791" src="https://github.com/user-attachments/assets/1fc94ce4-2427-4f61-b1db-7d13fb96ff81" />
Shopro 商城高级版：UniApp 多平台商城的工程架构与技术选型
引子：一张“五端适配”清单背后的工程真相
“一款基于 uni-app 的前端模板商城，适配了小程序 + App + H5 + 公众号”——这句话在产品介绍中经常出现，但它掩盖了一个事实：五端适配不是“写一次代码，编译五次”这么简单。

微信公众号、微信小程序、H5 网页、Android App、iOS App，这五个平台在运行环境、支付能力、登录方式、分享机制、页面路由、性能特征上都有本质差异。UniApp 解决的是“代码复用”的问题，但解决不了“平台差异”的问题。一套代码要同时跑在这五个平台上，需要在“统一”和“分化”之间找到精确的工程平衡。

本文以 Shopro 商城高级版为例，从多平台架构的适配策略、登录与支付的统一抽象、分享海报的跨端实现、路由拦截与登录态管理、多端状态同步，到商城核心功能的工程实现，展开这类系统的完整技术分析。

一、五端的本质差异
1.1 运行环境的差异
五个平台的运行环境差异，可以归纳为三个维度。

JavaScript 引擎。 微信公众号和小程序运行在微信的 JavaScript 引擎中，有独立的渲染层和逻辑层，不支持 DOM 操作。H5 运行在浏览器中，有完整的 DOM 和 BOM。App 运行在原生容器中，UniApp 使用原生渲染或 WebView 渲染。

平台能力。 微信小程序可以直接调用微信登录、微信支付、订阅消息、分享到聊天等原生能力。公众号可以调用微信 JS-SDK 的部分能力（分享、拍照、定位）。H5 无法直接调用微信能力（除非在微信内置浏览器中）。App 可以调用设备的所有原生能力（相机、相册、通讯录、推送）。

支付方式。 小程序使用 wx.requestPayment，公众号使用 WeixinJSBridge，H5 使用微信 H5 支付或支付宝网页支付，App 使用微信/支付宝原生 SDK。四种支付方式的参数构造和回调处理完全不同。

1.2 统一的能力抽象层
Shopro 的工程实践是：在业务代码和平台 API 之间建立一个能力抽象层。业务代码调用统一接口，抽象层根据平台分发到具体实现。

javascript
// utils/platform/index.js — 平台能力抽象层

class PlatformAdapter {
  /**
   * 登录：获取登录凭证
   */
  async getLoginCredential() {
    // #ifdef MP-WEIXIN
    return this.getWechatMiniCredential();
    // #endif

    // #ifdef H5
    if (this.isWechatBrowser()) {
      return this.getWechatOAuthCredential();
    }
    return this.getH5Credential();
    // #endif

    // #ifdef APP-PLUS
    return this.getAppCredential();
    // #endif
  }

  // #ifdef MP-WEIXIN
  getWechatMiniCredential() {
    return new Promise((resolve, reject) => {
      uni.login({
        provider: 'weixin',
        success: (res) => resolve({ type: 'wx_code', code: res.code }),
        fail: reject,
      });
    });
  }
  // #endif

  // #ifdef H5
  isWechatBrowser() {
    return /MicroMessenger/i.test(navigator.userAgent);
  }

  getWechatOAuthCredential() {
    const redirectUri = encodeURIComponent(window.location.href);
    const appid = this.getConfig('wechat_oa_appid');
    window.location.href = `https://open.weixin.qq.com/connect/oauth2/authorize?appid=${appid}&redirect_uri=${redirectUri}&response_type=code&scope=snsapi_userinfo&state=STATE#wechat_redirect`;
    return new Promise(() => {});
  }

  getH5Credential() {
    return Promise.resolve({ type: 'h5', needSms: true });
  }
  // #endif

  // #ifdef APP-PLUS
  getAppCredential() {
    return Promise.resolve({ type: 'app', needSms: true });
  }
  // #endif

  /**
   * 支付：统一支付接口
   */
  async pay(orderNo, payType) {
    const res = await uni.request({
      url: '/api/pay/create',
      method: 'POST',
      data: { order_no: orderNo, pay_type: payType },
    });

    const params = res.data.data;

    // #ifdef MP-WEIXIN
    if (payType === 'wechat') return this.payWechatMini(params);
    // #endif

    // #ifdef H5
    if (payType === 'wechat') return this.payWechatH5(params);
    if (payType === 'alipay') return this.payAlipayH5(params);
    // #endif

    // #ifdef APP-PLUS
    if (payType === 'wechat') return this.payWechatApp(params);
    if (payType === 'alipay') return this.payAlipayApp(params);
    // #endif

    throw new Error('当前平台不支持该支付方式');
  }

  // #ifdef MP-WEIXIN
  payWechatMini(params) {
    return new Promise((resolve, reject) => {
      uni.requestPayment({
        provider: 'wxpay',
        timeStamp: params.timeStamp,
        nonceStr: params.nonceStr,
        package: params.package,
        signType: params.signType,
        paySign: params.paySign,
        success: resolve,
        fail: reject,
      });
    });
  }
  // #endif

  // #ifdef H5
  payWechatH5(params) {
    window.location.href = params.mweb_url;
    return new Promise(() => {});
  }

  payAlipayH5(params) {
    window.location.href = params.pay_url;
    return new Promise(() => {});
  }
  // #endif

  // #ifdef APP-PLUS
  payWechatApp(params) {
    return new Promise((resolve, reject) => {
      uni.requestPayment({
        provider: 'wxpay',
        orderInfo: params,
        success: resolve,
        fail: reject,
      });
    });
  }

  payAlipayApp(params) {
    return new Promise((resolve, reject) => {
      uni.requestPayment({
        provider: 'alipay',
        orderInfo: params.orderInfo,
        success: resolve,
        fail: reject,
      });
    });
  }
  // #endif
}

export const platform = new PlatformAdapter();
业务代码只需要调用 platform.pay(orderNo, 'wechat')，不需要关心当前是哪个平台。 平台差异被封装在抽象层内部，通过条件编译分发到具体实现。

二、登录与用户体系
2.1 五种登录方式
Shopro 商城高级版支持五种登录方式：微信小程序登录、公众号网页授权登录、H5 手机号验证码登录、App 手机号验证码登录、App 微信登录。

javascript
// api/auth.js

export async function login(method, params = {}) {
  switch (method) {
    case 'wx_mini':
      // 小程序登录
      const { code } = await platform.getWechatMiniCredential();
      return uni.request({
        url: '/api/auth/wx_mini',
        method: 'POST',
        data: { code },
      });

    case 'wx_oa':
      // 公众号登录
      const { code: oaCode } = await platform.getWechatOAuthCredential();
      return uni.request({
        url: '/api/auth/wx_oa',
        method: 'POST',
        data: { code: oaCode },
      });

    case 'sms':
      // 手机号验证码登录
      return uni.request({
        url: '/api/auth/sms',
        method: 'POST',
        data: {
          phone: params.phone,
          code: params.code,
        },
      });

    case 'password':
      // 账号密码登录
      return uni.request({
        url: '/api/auth/password',
        method: 'POST',
        data: {
          account: params.account,
          password: params.password,
        },
      });
  }
}
2.2 Token 的自动刷新
登录态管理的核心问题是 Token 过期后的自动刷新。当请求返回 401 时，系统需要自动刷新 Token 并重试原请求。

javascript
// utils/request.js

let isRefreshing = false;
let pendingRequests = [];

async function request(options) {
  const token = uni.getStorageSync('token');
  if (token) {
    options.header = options.header || {};
    options.header['Authorization'] = `Bearer ${token}`;
  }

  return new Promise((resolve, reject) => {
    uni.request({
      ...options,
      success: async (res) => {
        // Token 过期，尝试刷新
        if (res.statusCode === 401) {
          if (!isRefreshing) {
            isRefreshing = true;
            try {
              await refreshToken();
              pendingRequests.forEach(cb => cb());
              pendingRequests = [];
            } catch (e) {
              uni.navigateTo({ url: '/pages/login/login' });
            } finally {
              isRefreshing = false;
            }
          }

          return new Promise((resolveRetry) => {
            pendingRequests.push(async () => {
              const retryRes = await request(options);
              resolveRetry(retryRes);
            });
          }).then(resolve).catch(reject);
        }

        if (res.statusCode === 200) {
          resolve(res.data);
        } else {
          reject(res);
        }
      },
      fail: reject,
    });
  });
}

async function refreshToken() {
  const refreshToken = uni.getStorageSync('refresh_token');
  if (!refreshToken) throw new Error('无 refresh token');

  const res = await new Promise((resolve, reject) => {
    uni.request({
      url: '/api/auth/refresh',
      method: 'POST',
      data: { refresh_token: refreshToken },
      success: resolve,
      fail: reject,
    });
  });

  if (res.data.code !== 200) throw new Error('刷新失败');

  uni.setStorageSync('token', res.data.data.token);
  uni.setStorageSync('refresh_token', res.data.data.refresh_token);
  uni.setStorageSync('token_expire_at', res.data.data.expire_at);
}
请求队列 + 单次刷新是核心设计。当多个请求同时返回 401 时，只触发一次刷新，其他请求等待刷新完成后重试。这避免了“多个请求同时刷新 Token”导致的竞态条件。

三、分享海报：Canvas 绘制的跨端实现
3.1 分享海报的工程挑战
分享海报是商城系统中最容易被低估的功能。它的技术挑战在于：Canvas 绘制在不同平台上的 API 差异极大。

小程序使用 Canvas 2D，H5 使用标准的 CanvasRenderingContext2D，App 使用 uni.createCanvasContext。图片的加载、绘制、导出也各有不同。

3.2 跨端海报绘制
javascript
// utils/poster.js

export async function generatePoster(options) {
  const {
    canvasId, width = 750, height = 1334,
    background, qrcode, avatar, nickname,
    productImage, productName, productPrice,
  } = options;

  const ctx = await getCanvasContext(canvasId, width, height);

  // 1. 绘制背景
  if (background) {
    const bg = await loadImage(background);
    ctx.drawImage(bg, 0, 0, width, height);
  } else {
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(0, 0, width, height);
  }

  // 2. 绘制商品图片（圆角）
  const prodImg = await loadImage(productImage);
  drawRoundedImage(ctx, prodImg, 40, 40, 670, 670, 20);

  // 3. 绘制商品名称
  ctx.fillStyle = '#333333';
  ctx.font = 'bold 32px sans-serif';
  wrapText(ctx, productName, 40, 760, 670, 44, 2);

  // 4. 绘制价格
  ctx.fillStyle = '#fe2c55';
  ctx.font = 'bold 48px sans-serif';
  ctx.fillText(`¥${productPrice}`, 40, 900);

  // 5. 绘制头像（圆形）
  if (avatar) {
    const avatarImg = await loadImage(avatar);
    drawCircleImage(ctx, avatarImg, 40, 1000, 80);
  }

  // 6. 绘制昵称
  ctx.fillStyle = '#666666';
  ctx.font = '28px sans-serif';
  ctx.fillText(nickname || '匿名用户', 140, 1050);

  // 7. 绘制二维码
  const qrImg = await loadImage(qrcode);
  ctx.drawImage(qrImg, 550, 980, 160, 160);

  // 8. 导出图片（分平台）
  return await exportCanvas(canvasId, width, height);
}

async function getCanvasContext(canvasId, width, height) {
  // #ifdef MP-WEIXIN
  return new Promise((resolve) => {
    const query = uni.createSelectorQuery();
    query.select(`#${canvasId}`)
      .fields({ node: true, size: true })
      .exec((res) => {
        const canvas = res[0].node;
        canvas.width = width;
        canvas.height = height;
        resolve(canvas.getContext('2d'));
      });
  });
  // #endif

  // #ifdef H5
  return new Promise((resolve) => {
    const canvas = document.getElementById(canvasId);
    canvas.width = width;
    canvas.height = height;
    resolve(canvas.getContext('2d'));
  });
  // #endif

  // #ifdef APP-PLUS
  return uni.createCanvasContext(canvasId);
  // #endif
}

function loadImage(src) {
  return new Promise((resolve, reject) => {
    // #ifdef MP-WEIXIN
    const img = uni.createOffscreenCanvas().createImage();
    img.onload = () => resolve(img);
    img.onerror = reject;
    img.src = src;
    // #endif

    // #ifdef H5
    const img = new Image();
    img.crossOrigin = 'anonymous';
    img.onload = () => resolve(img);
    img.onerror = reject;
    img.src = src;
    // #endif

    // #ifdef APP-PLUS
    uni.getImageInfo({
      src,
      success: (res) => resolve(res.path),
      fail: reject,
    });
    // #endif
  });
}

function drawRoundedImage(ctx, img, x, y, w, h, r) {
  ctx.save();
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.arcTo(x + w, y, x + w, y + h, r);
  ctx.arcTo(x + w, y + h, x, y + h, r);
  ctx.arcTo(x, y + h, x, y, r);
  ctx.arcTo(x, y, x + w, y, r);
  ctx.closePath();
  ctx.clip();
  ctx.drawImage(img, x, y, w, h);
  ctx.restore();
}

function drawCircleImage(ctx, img, x, y, size) {
  ctx.save();
  ctx.beginPath();
  ctx.arc(x + size / 2, y + size / 2, size / 2, 0, Math.PI * 2);
  ctx.closePath();
  ctx.clip();
  ctx.drawImage(img, x, y, size, size);
  ctx.restore();
}

function wrapText(ctx, text, x, y, maxWidth, lineHeight, maxLines) {
  const chars = text.split('');
  let line = '';
  let lineCount = 0;

  for (let i = 0; i < chars.length; i++) {
    const testLine = line + chars[i];
    const metrics = ctx.measureText(testLine);
    if (metrics.width > maxWidth && line !== '') {
      ctx.fillText(line, x, y + lineCount * lineHeight);
      line = chars[i];
      lineCount++;
      if (lineCount >= maxLines) {
        ctx.fillText('...', x, y + lineCount * lineHeight);
        return;
      }
    } else {
      line = testLine;
    }
  }
  ctx.fillText(line, x, y + lineCount * lineHeight);
}

async function exportCanvas(canvasId, width, height) {
  // #ifdef MP-WEIXIN
  return new Promise((resolve) => {
    uni.createSelectorQuery()
      .select(`#${canvasId}`)
      .fields({ node: true })
      .exec((res) => {
        const canvas = res[0].node;
        uni.canvasToTempFilePath({
          canvas,
          success: (r) => resolve(r.tempFilePath),
        });
      });
  });
  // #endif

  // #ifdef H5
  const canvas = document.getElementById(canvasId);
  return canvas.toDataURL('image/png');
  // #endif

  // #ifdef APP-PLUS
  return new Promise((resolve) => {
    uni.canvasToTempFilePath({
      canvasId,
      success: (r) => resolve(r.tempFilePath),
    });
  });
  // #endif
}
绘制逻辑在所有平台共享，只有 Canvas 上下文的获取和图片的导出是分平台的。

四、路由拦截与登录态管理
4.1 路由拦截的实现
UniApp 没有原生的路由守卫，需要通过 uni.addInterceptor 实现。

javascript
// utils/router/interceptor.js

const whiteList = [
  '/pages/index/index',
  '/pages/goods/detail',
  '/pages/goods/list',
  '/pages/login/login',
];

const needAuth = [
  '/pages/order/create',
  '/pages/order/list',
  '/pages/user/profile',
  '/pages/user/address',
  '/pages/cart/index',
];

export function setupRouterInterceptor() {
  uni.addInterceptor('navigateTo', {
    invoke(args) { return checkRoute(args.url); },
  });

  uni.addInterceptor('redirectTo', {
    invoke(args) { return checkRoute(args.url); },
  });

  uni.addInterceptor('switchTab', {
    invoke(args) { return checkRoute(args.url); },
  });
}

function checkRoute(url) {
  const path = url.split('?')[0];

  if (whiteList.some(w => path.startsWith(w))) return true;

  if (needAuth.some(a => path.startsWith(a))) {
    if (!isLoggedIn()) {
      uni.navigateTo({
        url: `/pages/login/login?redirect=${encodeURIComponent(url)}`,
      });
      return false;
    }
  }

  return true;
}

function isLoggedIn() {
  const token = uni.getStorageSync('token');
  if (!token) return false;

  const expireAt = uni.getStorageSync('token_expire_at');
  if (expireAt && expireAt < Date.now()) {
    uni.removeStorageSync('token');
    return false;
  }

  return true;
}
五、多端状态同步
5.1 购物车的本地缓存 + 服务端同步
购物车是需要在多端保持一致的核心数据。用户在手机小程序加入购物车，在 App 中打开应该看到相同的商品。

javascript
// store/cart.js

import { defineStore } from 'pinia';

export const useCartStore = defineStore('cart', {
  state: () => ({
    items: [],
    loading: false,
  }),

  getters: {
    totalCount: (state) => state.items.reduce((sum, i) => sum + i.quantity, 0),
    totalPrice: (state) => state.items.reduce((sum, i) => sum + i.price * i.quantity, 0),
  },

  actions: {
    async loadCart() {
      this.loading = true;
      try {
        // 先加载本地缓存，快速渲染
        const cached = uni.getStorageSync('cart_cache');
        if (cached && Date.now() - cached.timestamp < 3600_000) {
          this.items = cached.items;
        }

        // 再从服务端同步
        const { data } = await uni.request({ url: '/api/cart' });
        this.items = data.items;

        uni.setStorageSync('cart_cache', {
          items: this.items,
          timestamp: Date.now(),
        });
      } finally {
        this.loading = false;
      }
    },

    async addItem(product, quantity = 1) {
      // 乐观更新
      const existing = this.items.find(i => i.productId === product.id);
      if (existing) {
        existing.quantity += quantity;
      } else {
        this.items.push({
          productId: product.id,
          name: product.name,
          price: product.price,
          image: product.image,
          quantity,
          selected: true,
        });
      }

      try {
        await uni.request({
          url: '/api/cart/add',
          method: 'POST',
          data: { product_id: product.id, quantity },
        });
        uni.setStorageSync('cart_cache', {
          items: this.items,
          timestamp: Date.now(),
        });
      } catch (error) {
        await this.loadCart();
        throw error;
      }
    },
  },
});
乐观更新 + 回滚是核心设计。添加商品到购物车时，先更新本地状态让用户立即看到反馈，再发送请求。如果请求失败，从服务端重新加载购物车状态。

六、边界与张力
Shopro 商城高级版在工程实践中，有几个需要清醒认识的边界。

第一，五端一致性的测试成本。 五个平台的组合意味着测试矩阵是单端的五倍。每次功能迭代都需要在五个平台上验证。虽然 UniApp 的条件编译降低了代码维护成本，但没有降低测试成本。

第二，小程序包体积限制。 微信小程序主包限制 2MB，总包限制 20MB。商城系统的图片、字体、第三方库都会占用包体积。工程实践包括：图片使用 CDN、分包加载、按需引入组件。

第三，分享海报的图片跨域问题。 H5 端绘制海报时，如果商品图片来自不同的域名，Canvas 会因为跨域污染而无法导出。解决方案是：图片使用同源域名、配置 CORS 头，或通过后端代理转发图片。

第四，多支付方式的回调一致性。 微信支付、支付宝支付、H5 支付的回调格式完全不同。系统需要为每种支付方式实现独立的回调解析逻辑，但最终都归一到统一的“订单支付成功”事件。

第五，App 端的支付合规性。 微信和支付宝对 App 支付的接入有严格的审核要求。虚拟商品在 iOS 上禁止使用第三方支付，系统需要在 App 端做特殊处理。

结语：多平台商城的工程本质是“统一抽象 + 差异隔离”
Shopro 商城高级版的核心工程挑战，可以归结为一句话：在统一业务逻辑的同时，正确隔离平台差异。

登录、支付、分享、Canvas 绘制这些平台相关的操作，都被封装在能力抽象层内部，业务代码只调用统一接口。条件编译是实现这层隔离的工具，但应该谨慎使用——能用运行时检测解决的，不要用编译时分支。

这套系统的价值，最终体现在一个朴素的判断上：用户从微信公众号、微信小程序、H5 网页、Android App 还是 iOS App 进入商城，看到的商品、购物车、订单和积分，应该是完全一致的。 这个“一致”就是多平台商城系统的工程底线。
