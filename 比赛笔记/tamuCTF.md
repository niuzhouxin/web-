## bad-apple
题目给了附件Dockerfile和配置文件，读一下可以知道
创建了这几个目录
```
RUN mkdir -p /srv/app /srv/http/uploads/admin /srv/http/static/frames/shared/bad-apple
```
并且
```
CMD ["sh", "-c", "HEX=$(openssl rand -hex 16) && mv /srv/http/uploads/admin/flag.gif /srv/http/uploads/admin/$HEX-flag.gif && echo $HEX > /srv/http/.flag_secret && httpd -DFOREGROUND"]
```
得知flag就藏在`/srv/http/uploads/admin/`目录下。
再看配置文件
```
Alias /browse /srv/http/uploads
```
这是把`/srv/http/uploads`这个目录映射到`/browse`目录下了，所以可以直接访问`/browse`
再看一下源码
```python
@app.route('/convert')  
def convert():  
    user_id = request.args.get('user_id', 'anonymous')  
    filename = request.args.get('filename', '')  
  
    input_path = os.path.join(app.config['UPLOAD_FOLDER'], secure_filename(user_id), filename)  
    if not os.path.exists(input_path):  
        return "File not found", 404  
  
    safe_name = os.path.splitext(os.path.basename(filename))[0]  
    output_dir = os.path.join(FRAMES_BASE, user_id, safe_name)  
    os.makedirs(output_dir, exist_ok=True)  
  
    try:  
        frame_count = extract_frames(input_path, output_dir, safe_name)  
        return redirect(url_for('index', view=safe_name, user_id=user_id))  
    except Exception as e:  
        return f"Error processing file: {str(e)}", 500
```
发现这里的`filename`没有进行任何过滤，而`user_id`是过滤了的。
但是直接访问`/browse/admin/e017b6321bda6812ec80e9fac368709e-flag.gif`是不被允许的。
所以要用到`/get_frames`路由。访问`/get_frames?user_id=solver&gif_name=e017b6321bda6812ec80e9fac368709e-flag`，这样就可以把gif动图拆分成一个个png图片。
