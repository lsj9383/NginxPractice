# 快速开始

## 完整示例

假设我们要开发的 http 模块名称为 `ngx_http_test_module`。

- 目录结构

  ```text
  ├─nginx-1.19.4
  └─ngx_http_test_module
    ├─config
    └─ngx_http_test_module.c
  ```

- 编译

  ```sh
  $ cd nginx-1.19.4
  $ ./configure --prefix=/root/workspace/ngx-test-env/nginx \
               --with-http_ssl_module \
               --add-module=../ngx_http_test_module
  $ make && make install
  ```

- `ngx_http_test_module/config`

  ```sh
  # 必须和 ngx_http_test_module/ngx_http_test_module.c 中的 ngx_module_t 模块变量名一致
  ngx_addon_name=ngx_http_test_module

  HTTP_MODULES="$HTTP_MODULES ${ngx_addon_name}"
  NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_test_module.c"
  ```

- `ngx_http_test_module/ngx_http_test_module.c`

  ```c
  #include <ngx_config.h>
  #include <ngx_core.h>
  #include <ngx_http.h>
  
  static ngx_int_t ngx_http_test_post_config(ngx_conf_t *cf);
  static void* ngx_http_test_create_loc_conf(ngx_conf_t *cf);
  
  static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r);
  
  typedef struct {
      ngx_str_t                      test_str;
  } ngx_http_test_conf_t;
  
  
  static ngx_command_t ngx_http_test_commands[] = {
      ngx_null_command
  };
  
  static ngx_http_module_t ngx_http_test_module_ctx = {
      NULL,
      ngx_http_test_post_config,
      NULL,
      NULL,
      NULL,
      NULL,
      ngx_http_test_create_loc_conf,
      NULL
  };

  // 模块变量名必须和 config 中的 ngx_addon_name 一致
  ngx_module_t ngx_http_test_module = {
      NGX_MODULE_V1,
      &ngx_http_test_module_ctx,
      ngx_http_test_commands,
      NGX_HTTP_MODULE,
      NULL,
      NULL,
      NULL,
      NULL,
      NULL,
      NULL,
      NULL,
      NGX_MODULE_V1_PADDING
  };
  
  static void * ngx_http_test_create_loc_conf(ngx_conf_t *cf)
  {
      ngx_http_test_conf_t  *conf;
  
      conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_test_conf_t));
      if (conf == NULL) {
          return NULL;
      }
  
      return conf;
  }
  
  static ngx_int_t ngx_http_test_post_config(ngx_conf_t *cf) {
      ngx_http_handler_pt             *h;
      ngx_http_core_main_conf_t       *cmcf;
  
  
      cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
  
      h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
      if (h == NULL) {
          return NGX_ERROR;
      }
  
      *h = ngx_http_test_handler;
  
      return NGX_OK;
  }
  
  static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r) {
      return NGX_DECLINED;
  }
  ```
