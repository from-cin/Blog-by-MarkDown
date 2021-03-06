---
title: TP QINIU
date: 2021-08-06 18:30:42
tags:
---

1. 首先我们引入七牛的插件

   ```
   composer require qiniu/php-sdk	--tp
   composer require itbdw/laravel-storage-qiniu  --laravel
   ```

2. 在配置文件中配置七牛云的参数

   ```
   tp的app.php中加上
   'qiniu' => [
           'accesskey' => '你的accesskey',
           'secretkey' => '你的secretkey',
           'bucket'    => '存储空间',
           'domain'    =>  '域名'
       ]
       
        laravel 中 config/app.php 里面的 providers 加上一行	itbdw\QiniuStorage\QiniuFilesystemServiceProvider::class,config/filesystems.php 里面的 disks数组加上
         'qiniu' => [
               'driver'    => 'qiniu',
               'domain'    => '88888888.bkt.clouddn.com',  //你的七牛域名
               'access_key'=> '',    //AccessKey
               'secret_key'=> '',   //SecretKey
               'bucket'    => 'wl11',    //Bucket名字你的储存空间的名字
           ],
   ```

3. 前端使用webuploader上传

   ```
   function Upload(i,num){
       sumimg(num);
   //            var token = 'ZTEpdpZn1J1fLzgTfNtLXvRmcu0bjcnTUUhoQsEW';
       var uploader = WebUploader.create({
           auto: true,
           swf: '/static/extends/lib/webuploader/0.1.5/Uploader.swf',//文件接收服务端。
           server: '/admin/tool/uploadImg',//文件接收服务端。
           pick: '#filePicker'+i,//指定选择文件的按钮容器
           fileNumLimit: num,//文件数量
           compress: {
               // 图片质量，只有type为`image/jpeg`的时候才有效。
               quality: 90,
           },
           accept: { //指定接受哪些类型的文件
               title: 'Images',
               extensions: 'gif,jpg,jpeg,bmp,png',
               mimeTypes: 'image/*'
           }
       });
       uploader;
       //当文件被加入队列以后触发
       uploader.on('fileQueued',function(file){
           var $li = $(
                   '<div id="' + file.id + '" class="file-item thumbnail" onclick="delUpload(this,'+i+','+num+');">' +
                   '<input type="hidden" name="img[]" id="'+file.id+'id" />'+
                   '<img>' +
                   '<div class="info">开始上传图片...</div>' +
                   '</div>'
               ),
               $img = $li.find('img');
           $("#fileList"+i).append($li);//图片添加到容器
           var sum = $("#fileList"+i).find('.file-item').length;
           if(sum >= num){
               $("#filePicker"+i).attr('style','display:none;');
           }
           // 创建缩略图显示
           uploader.makeThumb(file,function(error,src){
               if (error){
                   $img.replaceWith('<span>不能预览</span>');
                   return;
               }
               $img.attr('src',src);
           },200,200);
       });
       // 文件上传过程中创建进度百分比实时显示。
       uploader.on('uploadProgress',function(file,percentage){
           $("#"+file.id+" .info").html('正在上传（'+parseInt(percentage*100)+'%）');
       });
   
       //文件上传返回成功，判断是否成功上传到七牛
       uploader.on('uploadSuccess', function(file,response){
           if(response.code == 0){
               $("#"+file.id+" .info").html('上传失败');
           }else{
               sumimg(num);
               $("#"+file.id+" .info").html('上传完成,点击图片可删除');
               $( '#'+file.id ).addClass('upload-state-done');
               $( '#'+file.id+"id").val(response.data);//赋值到input
           }
       });
   }
   ```

4. 后台代码

       // 初始化签权对象
       $auth = new Auth($this->accessKey,$this->secretKey);
       $token = $auth->uploadToken($this->_bucket);
       
       // 构建 UploadManager 对象
       $uploadMrg = new UploadManager();
       
       // 上传文件到七牛
       $files = $_FILES;
       $values = array_values($files);
       $saveName = hash_file('sha1', $values[0]['tmp_name']) . time();
       list($ret, $err) = $uploadMrg->putFile($token, $saveName, $values[0]['tmp_name']);
       
       if ($err !== null) {
           $return =  [
               'code'  =>  0,
               'msg'   =>  '上传图片失败'
           ];
       } else {
           $return =  [
               'code'  =>  1,
               'msg'   =>  '上传成功',
               'data'  =>  'http://' . $this->_domain . '/' . $ret['key']
           ];
       }
       return json($return);

5. 删除图片$img = Request::param('name');

       $imgArr = explode('/',$img);
       
       $delImgName = end($imgArr);
       
           // 初始化签权对象
       $auth = new Auth($this->_accessKey,$this->_secretKey);
       
       $config = new Config();
       //        管理资源
       
           $bucketManager = new BucketManager($auth,$config);
       
       //    	  删除文件操作
       
           $res = $bucketManager -> delete($this->_bucket,$delImgName);
           if (is_null($res) || $res == null) {
                   // 为null成功
                   $return = [
                       'code'  =>  1,
                       'msg'   =>  '删除成功'
                   ];
               }else{
                   $return = [
                       'code'  =>  0,
                       'msg'   =>  '删除失败'
                   ];
               }
       
               return json($return);

6. 还有一种更简单的方式(后台操作)直接使用开发好的插件这里是使用的海豚php(和tp差不多,laravel参照即可)

   ```
   <?php
   // +----------------------------------------------------------------------
   // | 海豚PHP框架 [ DolphinPHP ]
   // +----------------------------------------------------------------------
   // | 版权所有 2016~2017 河源市卓锐科技有限公司 [ http://www.zrthink.com ]
   // +----------------------------------------------------------------------
   // | 官方网站: http://dolphinphp.com
   // +----------------------------------------------------------------------
   // | 开源协议 ( http://www.apache.org/licenses/LICENSE-2.0 )
   // +----------------------------------------------------------------------
   
   namespace plugins\QiNiu;
   
   
   use app\common\controller\Plugin;
   use app\admin\model\Attachment as AttachmentModel;
   use think\Db;
   use think\Image;
   use Qiniu\Auth;
   use Qiniu\Storage\UploadManager;
   use think\facade\Env;
   require Env::get('root_path').'plugins/QiNiu/SDK/autoload.php';
   
   /**
    * 七牛上传插件
    * @package plugins\QiNiu
    * @author 蔡伟明 <314013107@qq.com>
    */
   class QiNiu extends Plugin
   {
       /**
        * @var array 插件信息
        */
       public $info = [
           // 插件名[必填]
           'name'        => 'QiNiu',
           // 插件标题[必填]
           'title'       => '七牛上传插件',
           // 插件唯一标识[必填],格式：插件名.开发者标识.plugin
           'identifier'  => 'qi_niu.ming.plugin',
           // 插件图标[选填]
           'icon'        => 'fa fa-fw fa-upload',
           // 插件描述[选填]
           'description' => '仅支持DolphinPHP1.0.6以上版本，安装后，需将【<a href="/admin.php/admin/system/index/group/upload.html">上传驱动</a>】将其设置为“七牛云”。在附件管理中删除文件，并不会同时删除七牛云上的文件。',
           // 插件作者[必填]
           'author'      => '蔡伟明',
           // 作者主页[选填]
           'author_url'  => 'http://www.caiweiming.com',
           // 插件版本[必填],格式采用三段式：主版本号.次版本号.修订版本号
           'version'     => '1.1.0',
           // 是否有后台管理功能[选填]
           'admin'       => '0',
       ];
   
       /**
        * @var array 插件钩子
        */
       public $hooks = [
           'upload_attachment'
       ];
   
       /**
        * 上传附件
        * @param array $params 参数
        * @author 蔡伟明 <314013107@qq.com>
        * @return mixed
        */
       public function uploadAttachment($params = [])
       {
           $file = $params['file'];
           // 缩略图参数
           $thumb = request()->post('thumb', '');
           // 水印参数
           $watermark = request()->post('watermark', '');
   
           $config = $this->getConfigValue();
   
           $error_msg = '';
           if ($config['ak'] == '') {
               $error_msg = '未填写七牛【AccessKey】';
           } elseif ($config['sk'] == '') {
               $error_msg = '未填写七牛【SecretKey】';
           } elseif ($config['bucket'] == '') {
               $error_msg = '未填写七牛【Bucket】';
           } elseif ($config['domain'] == '') {
               $error_msg = '未填写七牛【Domain】';
           }
           if ($error_msg != '') {
               switch ($params['from']) {
                   case 'wangeditor':
                       return "error|{$error_msg}";
                       break;
                   case 'ueditor':
                       return json(['state' => $error_msg]);
                       break;
                   case 'editormd':
                       return json(["success" => 0, "message" => $error_msg]);
                       break;
                   case 'ckeditor':
                       return ck_js(request()->get('CKEditorFuncNum'), '', $error_msg);
                       break;
                   default:
                       return json([
                           'code'   => 0,
                           'class'  => 'danger',
                           'info'   => $error_msg
                       ]);
               }
           }
   
           $config['domain'] = rtrim($config['domain'], '/').'/';
   
           // 移动到框架应用根目录/uploads/ 目录下
           $info = $file->move(config('upload_path') . DIRECTORY_SEPARATOR . 'temp', '');
           // 文件信息
           $file_info = $file->getInfo();
           // 要上传文件的本地路径
           $filePath = $info->getPathname();
           // 上传到七牛后保存的文件名
           $file_name = explode('.', $file_info['name']);
           $ext = end($file_name);
           $key = $info->hash('md5').'.'.$ext;
   
           // 如果是图片，则获取宽度和高度，并判断是否添加水印
           $ext_limit = config('upload_image_ext');
           $ext_limit = $ext_limit == '' ? [] : explode(',', $ext_limit);
           // 缩略图路径
           $thumb_path_name = '';
           if (preg_grep("/$ext/i", $ext_limit)) {
               $img = Image::open($info);
               $img_width  = $img->width();
               $img_height = $img->height();
   
               // 水印功能
               if ($watermark == '') {
                   if (config('upload_thumb_water') == 1 && config('upload_thumb_water_pic') > 0) {
                       $this->create_water($info->getRealPath(), config('upload_thumb_water_pic'));
                   }
               } else {
                   if (strtolower($watermark) != 'close') {
                       list($watermark_img, $watermark_pos, $watermark_alpha) = explode('|', $watermark);
                       $this->create_water($info->getRealPath(), $watermark_img, $watermark_pos, $watermark_alpha);
                   }
               }
   
               // 生成缩略图
               if ($thumb == '') {
                   if (config('upload_image_thumb') != '') {
                       list($thumb_max_width, $thumb_max_height) = explode(',', config('upload_image_thumb'));
                       $thumb_path_name = $config['domain'].$key.'?imageMogr2/thumbnail/'.$thumb_max_width.'x'.$thumb_max_height;
                   }
               } else {
                   if (strtolower($thumb) != 'close') {
                       list($thumb_size, $thumb_type) = explode('|', $thumb);
                       list($thumb_max_width, $thumb_max_height) = explode(',', $thumb_size);
                       $thumb_path_name = $config['domain'].$key.'?imageMogr2/thumbnail/'.$thumb_max_width.'x'.$thumb_max_height;
                   }
               }
           } else {
               $img_width  = '';
               $img_height = '';
           }
   
           // 需要填写你的 Access Key 和 Secret Key
           $accessKey = $config['ak'];
           $secretKey = $config['sk'];
           // 构建鉴权对象
           $auth = new Auth($accessKey, $secretKey);
           // 要上传的空间
           $bucket = $config['bucket'];
           // 生成上传 Token
           $token = $auth->uploadToken($bucket, $key);
           // 初始化 UploadManager 对象并进行文件的上传
           $uploadMgr = new UploadManager();
           // 调用 UploadManager 的 putFile 方法进行文件的上传
           list($ret, $err) = $uploadMgr->putFile($token, $key, $filePath);
           if ($err !== null) {
               return json(['code' => 0, 'class' => 'danger', 'info' => '上传失败:'.$err]);
           } else {
               // 获取附件信息
               $data = [
                   'uid'    => session('user_auth.uid'),
                   'name'   => $file_info['name'],
                   'mime'   => $file_info['type'],
                   'path'   => $config['domain'].$key.'?v='.rand(111111, 999999),
                   'ext'    => $info->getExtension(),
                   'size'   => $info->getSize(),
                   'md5'    => $info->hash('md5'),
                   'sha1'   => $info->hash('sha1'),
                   'thumb'  => $thumb_path_name,
                   'module' => $params['module'],
                   'driver' => 'qiniu',
                   'width'  => $img_width,
                   'height' => $img_height,
               ];
               if ($file_add = AttachmentModel::create($data)) {
                   unset($info);
                   // 删除本地临时文件
                   @unlink(config('upload_path') . DIRECTORY_SEPARATOR . 'temp'.DIRECTORY_SEPARATOR.$file_info['name']);
                   // 返回结果
                   switch ($params['from']) {
                       case 'wangeditor':
                           return $data['path'];
                           break;
                       case 'ueditor':
                           return json([
                               "state" => "SUCCESS",          // 上传状态，上传成功时必须返回"SUCCESS"
                               "url"   => $data['path'], // 返回的地址
                               "title" => $file_info['name'], // 附件名
                           ]);
                           break;
                       case 'editormd':
                           return json([
                               "success" => 1,
                               "message" => '上传成功',
                               "url"     => $data['path'],
                           ]);
                           break;
                       case 'ckeditor':
                           return ck_js(request()->get('CKEditorFuncNum'), $data['path']);
                           break;
                       default:
                           return json([
                               'code'   => 1,
                               'info'   => '上传成功',
                               'class'  => 'success',
                               'id'     => $file_add['id'],
                               'path'   => $data['path']
                           ]);
                   }
               } else {
                   switch ($params['from']) {
                       case 'wangeditor':
                           return "error|上传失败";
                           break;
                       case 'ueditor':
                           return json(['state' => '上传失败']);
                           break;
                       case 'editormd':
                           return json(["success" => 0, "message" => '上传失败']);
                           break;
                       case 'ckeditor':
                           return ck_js(request()->get('CKEditorFuncNum'), '', '上传失败');
                           break;
                       default:
                           return json(['code' => 0, 'class' => 'danger', 'info' => '上传失败']);
                   }
               }
           }
       }
   
       /**
        * 添加水印
        * @param string $file 要添加水印的文件路径
        * @param string $watermark_img 水印图片id
        * @param string $watermark_pos 水印位置
        * @param string $watermark_alpha 水印透明度
        * @author 蔡伟明 <314013107@qq.com>
        */
       private function create_water($file = '', $watermark_img = '', $watermark_pos = '', $watermark_alpha = '')
       {
           $img  = Db::name('admin_attachment')->where('id', $watermark_img)->find();
           $path = $img['path'];
           $tmp  = false;
           if (strtolower(substr($path, 0, 4)) == 'http') {
               $file_watermark  = file_get_contents($path);
               $thumb_water_pic = config('upload_path') . DIRECTORY_SEPARATOR . 'temp/'.$img['md5'].'.'.$img['ext'];
               if (false === file_put_contents($thumb_water_pic, $file_watermark)) {
                   return;
               }
               $tmp = true;
           } else {
               $thumb_water_pic = realpath(Env::get('root_path') . 'public/' . $path);
           }
   
           if (is_file($thumb_water_pic)) {
               // 读取图片
               $image = Image::open($file);
               // 添加水印
               $watermark_pos   = $watermark_pos   == '' ? config('upload_thumb_water_position') : $watermark_pos;
               $watermark_alpha = $watermark_alpha == '' ? config('upload_thumb_water_alpha') : $watermark_alpha;
               $image->water($thumb_water_pic, $watermark_pos, $watermark_alpha);
               // 保存水印图片，覆盖原图
               $image->save($file);
               // 删除临时文件
               if ($tmp) {
                   unlink($thumb_water_pic);
               }
           }
       }
   
       /**
        * 安装方法
        * @author 蔡伟明 <314013107@qq.com>
        * @return bool
        */
       public function install(){
           if (!version_compare(config('dolphin.product_version'), '1.0.6', '>=')) {
               $this->error = '本插件仅支持DolphinPHP1.0.6或以上版本';
               return false;
           }
           $upload_driver = Db::name('admin_config')->where(['name' => 'upload_driver', 'group' => 'upload'])->find();
           if (!$upload_driver) {
               $this->error = '未找到【上传驱动】配置，请确认DolphinPHP版本是否为1.0.6以上';
               return false;
           }
           $options = parse_attr($upload_driver['options']);
           if (isset($options['qiniu'])) {
               $this->error = '已存在名为【qiniu】的上传驱动';
               return false;
           }
           $upload_driver['options'] .= PHP_EOL.'qiniu:七牛云';
   
           $result = Db::name('admin_config')
               ->where(['name' => 'upload_driver', 'group' => 'upload'])
               ->setField('options', $upload_driver['options']);
   
           if (false === $result) {
               $this->error = '上传驱动设置失败';
               return false;
           }
           return true;
       }
   
       /**
        * 卸载方法
        * @author 蔡伟明 <314013107@qq.com>
        * @return bool
        */
       public function uninstall(){
           $upload_driver = Db::name('admin_config')->where(['name' => 'upload_driver', 'group' => 'upload'])->find();
           if ($upload_driver) {
               $options = parse_attr($upload_driver['options']);
               if (isset($options['qiniu'])) {
                   unset($options['qiniu']);
               }
               $options = implode_attr($options);
               $result = Db::name('admin_config')
                   ->where(['name' => 'upload_driver', 'group' => 'upload'])
                   ->update(['options' => $options, 'value' => 'local']);
   
               if (false === $result) {
                   $this->error = '上传驱动设置失败';
                   return false;
               }
           }
           return true;
       }
   }
   ```

   



