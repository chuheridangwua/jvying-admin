<template>
    <div class="upload-carousel">
        <h1 style="margin-bottom: 20px;">首页轮播图</h1>
        <el-upload class="upload-demo" action="#" list-type="picture-card" :file-list="fileList"
            :on-preview="handlePreview" :on-remove="handleRemove" :before-upload="beforeUpload" multiple>
            <i class="el-icon-plus"></i>
        </el-upload>
        <el-dialog :visible.sync="previewVisible">
            <div class="image-preview"><img width="100%" :src="previewImage" alt=""></div>
        </el-dialog>
        <el-button type="primary" @click="uploadImages" style="margin-top: 20px;">上传/更新轮播图</el-button>
    </div>
</template>

<script>
import app from '@/api/appwx';
import db from '@/api/database';

export default {
    data() {
        return {
            fileList: [],
            previewVisible: false,
            previewImage: '',
        };
    },
    created() {
        this.fetchExistingImages();
    },
    methods: {
        async fetchExistingImages() {
            try {
                const res = await db.collection('carouselChart-chengjun').get();
                console.log(res);
                console.log(res.data[0].data.images);
                if (res.data.length > 0 && res.data[0].data.images && res.data[0].data.images.length > 0) {
                    const images = res.data[0].data.images; // 确保 images 数组非空
                    const downloadRes = await app.getTempFileURL({
                        fileList: images,
                    });
                    this.fileList = downloadRes.fileList.map(file => ({
                        name: file.fileID,
                        url: file.tempFileURL,
                    }));
                } else {
                    console.log('没有图片');
                }
            } catch (error) {
                console.error('获取图片失败:', error);
                this.$message.error('获取现有轮播图失败');
            }
        },
        beforeUpload(file) {
            const reader = new FileReader();
            reader.readAsDataURL(file);
            reader.onload = e => {
                // 手动添加一个isNew标记
                this.fileList.push({
                    name: file.name,
                    url: e.target.result,
                    isNew: true, // 标记为新文件
                });
            };
            return false;
        },
        handlePreview(file) {
            this.previewImage = file.url;
            this.previewVisible = true;
        },
        async handleRemove(file) {
            const index = this.fileList.findIndex(f => f.name === file.name);
            if (index !== -1) {
                // 从 fileList 中移除该文件
                this.fileList.splice(index, 1);
                // 更新数据库中的图片数组
                await this.updateDatabaseImages();
            }
        },
        async updateDatabaseImages() {
            try {
                const uploadedFileIDs = this.fileList.map(file => file.name); // 假设 file.name 存储的是图片在云存储中的 ID
                // 检查数据库中是否已有轮播图记录
                const res = await db.collection('carouselChart-chengjun').get();
                if (res.data.length > 0) {
                    // 如果已有记录，更新该记录
                    await db.collection('carouselChart-chengjun').doc(res.data[0]._id).update({
                        data: {
                            images: uploadedFileIDs,
                        }
                    });
                    console.log("数据库记录更新成功");
                    this.$message.success('轮播图更新成功');
                } else {
                    console.log("没有找到现有的轮播图记录来更新");
                    // 根据应用逻辑决定是否在这里创建新记录
                }
            } catch (error) {
                console.error("更新数据库过程中出现错误", error);
                this.$message.error('更新轮播图失败');
            }
        },
        async uploadImages() {
            console.log("开始上传图片...");
            console.log("this.fileList", this.fileList);

            // 使用isNew属性来筛选出新图片
            const newImages = this.fileList.filter(file => file.isNew);

            if (newImages.length === 0) {
                console.log("没有新图片需要上传");
                // 这里可以处理没有新图片但仍需要更新其他信息的逻辑
                return;
            }

            const uploadPromises = newImages.map((file, index) => {
                // 这里需要确保 cloudPath 不会因为错误地处理文件名而重复
                // 假设文件名已经安全处理，移除特殊字符
                const safeFileName = file.name.replace(/[^a-zA-Z0-9.]/g, "_");
                // 构建 cloudPath 时要避免重复拼接路径部分
                const cloudPath = `carouselImages/${Date.now()}-${index}-${safeFileName}`;

                const blob = this.dataURLtoBlob(file.url); // 将数据URL转换为Blob
                console.log(`准备上传图片：${file.name} 到 ${cloudPath}`);
                return app.uploadFile({
                    cloudPath: cloudPath,
                    filePath: blob, // 使用Blob对象进行上传
                }).then(res => {
                    console.log(`图片${file.name}上传成功，返回结果：`, res);
                    return res.fileID; // 上传后返回的结果中包含 fileID
                }).catch(error => {
                    console.error(`图片${file.name}上传失败，错误：`, error);
                    throw error;
                });
            });

            try {
                const uploadedFileIDs = await Promise.all(uploadPromises);
                console.log("新图片上传完成, 返回的 FileIDs：", uploadedFileIDs);

                // 检查数据库中是否已有轮播图记录
                const res = await db.collection('carouselChart-chengjun').get();
                if (res.data.length > 0) {
                    // 如果已有记录，更新该记录
                    const existingImages = res.data[0].data.images || [];
                    const newImageIds = uploadedFileIDs; // 假设这里只有新图片的ID
                    const updatedImages = existingImages.concat(newImageIds);

                    await db.collection('carouselChart-chengjun').doc(res.data[0]._id).update({
                        data: {
                            images: updatedImages,
                        }
                    });
                    console.log("数据库记录更新成功");
                } else {
                    // 如果没有记录，创建新记录
                    await db.collection('carouselChart-chengjun').add({
                        data: {
                            images: uploadedFileIDs,
                        }
                    });
                    console.log("新数据库记录创建成功");
                }
                this.$message.success('轮播图更新成功');
            } catch (error) {
                console.error("上传图片或更新数据库过程中出现错误", error);
                this.$message.error('轮播图更新失败');
            }
        },

        dataURLtoBlob(dataurl) {
            if (!dataurl || !dataurl.startsWith('data:')) {
                console.error('Invalid data URL passed to dataURLtoBlob:', dataurl);
                // 返回一个空的Blob对象或其他适当的默认值
                return new Blob([], { type: 'text/plain' });
            }
            var arr = dataurl.split(','), mime = arr[0].match(/:(.*?);/)[1],
                bstr = atob(arr[1]), n = bstr.length, u8arr = new Uint8Array(n);
            while (n--) {
                u8arr[n] = bstr.charCodeAt(n);
            }
            return new Blob([u8arr], { type: mime });
        },
    }
};
</script>

<style>
.el-upload-list__item-thumbnail {
    /* 图片在方框内显示长边 */
    object-fit: contain;
}


::v-deep .el-upload--picture-card,
::v-deep .el-upload-list__item {
    width: 100px;
    height: 100px;
    line-height: 110px;
}

.upload-carousel {
    margin: 50px;
}

.upload-demo i.el-icon-plus {
    font-size: 32px;
    color: #999;
}

.upload-demo .el-upload__input {
    display: none !important;
}

.tips {
    margin-top: 20px;
    font-size: 14px;
    color: #666;
}

.image-preview img {
    width: 100%;
    /* 或者 height: 100%; 根据需要选择 */
    object-fit: contain;
    /* 保持图片的原始长宽比，且保证图片完整显示 */
    border: 0;
    /* 如果不需要边框可以加上这一行 */
}
</style>