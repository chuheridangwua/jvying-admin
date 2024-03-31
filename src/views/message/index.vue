<template>
    <div>
        <el-button style="margin: 20px 30px 0px;" @click="handleNew" type="primary">新建消息</el-button>
        <!-- 消息列表展示 -->
        <el-table :data="messages" style="width: 100%;margin: 20px 30px;">
            <el-table-column prop="title" label="标题"></el-table-column>
            <el-table-column prop="content" label="内容"></el-table-column>
            <el-table-column prop="createdAt" label="创建时间"></el-table-column>
            <el-table-column label="操作">
                <template #default="scope">
                    <el-button size="mini" @click="handleEdit(scope.row)">编辑</el-button>
                    <el-button size="mini" type="danger" @click="handleDelete(scope.row.id)">删除</el-button>
                </template>
            </el-table-column>
        </el-table>

        <!-- 添加/编辑消息对话框 -->
        <el-dialog :visible.sync="dialogVisible" title="消息">
            <el-form :model="editForm">
                <el-form-item label="标题">
                    <el-input v-model="editForm.title"></el-input>
                </el-form-item>
                <el-form-item label="内容">
                    <el-input v-model="editForm.content"></el-input>
                </el-form-item>
            </el-form>
            <template #footer>
                <span class="dialog-footer">
                    <el-button @click="dialogVisible = false">取 消</el-button>
                    <el-button type="primary" @click="saveMessage">确 定</el-button>
                </span>
            </template>
        </el-dialog>

        <!-- 新建消息按钮 -->
    </div>
</template>

<script>
import app from '@/api/appwx';
import db from '@/api/database';

export default {
    data() {
        return {
            messages: [], // 消息列表数据
            dialogVisible: false, // 控制对话框显示
            editForm: { // 编辑表单数据
                id: '',
                title: '',
                content: ''
            },
            isEdit: false // 是否为编辑状态
        };
    },
    mounted() {
        this.fetchMessages();
    },
    methods: {
        // 获取消息列表
        fetchMessages() {
            db.collection('message').orderBy('data.createdAt', 'desc').get().then(res => {
                this.messages = res.data.map(doc => ({
                    id: doc._id,
                    ...doc.data
                }));
            });
        },
        // 添加消息
        addMessage() {
            const now = new Date();
            const createdAt = `${now.getFullYear()}-${(now.getMonth() + 1).toString().padStart(2, '0')}-${now.getDate().toString().padStart(2, '0')} ${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;

            db.collection('message').add({
                data: {
                    title: this.editForm.title,
                    content: this.editForm.content,
                    createdAt: createdAt
                }
            }).then(() => {
                this.dialogVisible = false; // 关闭对话框
                this.fetchMessages(); // 重新加载消息列表
                this.$message.success('消息添加成功');
            }).catch(err => {
                console.error("添加消息失败", err);
                this.$message.error('消息添加失败');
            });
        },
        // 更新消息
        updateMessage() {
            const now = new Date();
            const createdAt = `${now.getFullYear()}-${(now.getMonth() + 1).toString().padStart(2, '0')}-${now.getDate().toString().padStart(2, '0')} ${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;

            db.collection('message').doc(this.editForm.id).update({
                data: {
                    title: this.editForm.title,
                    content: this.editForm.content,
                    createdAt: createdAt
                }
            }).then(() => {
                this.dialogVisible = false; // 关闭对话框
                this.fetchMessages(); // 重新加载消息列表
            }).catch(err => {
                console.error("更新消息失败", err);
            });
        },
        // 删除消息
        deleteMessage(id) {
            db.collection('message').doc(id).remove().then(() => {
                this.fetchMessages(); // 重新加载消息列表
            }).catch(err => {
                console.error("删除消息失败", err);
            });
        },
        // 处理新建消息
        handleNew() {
            this.isEdit = false;
            this.editForm = { id: '', title: '', content: '' }; // 重置表单
            this.dialogVisible = true; // 显示对话框
        },
        // 处理编辑消息
        handleEdit(row) {
            this.isEdit = true;
            this.editForm = { id: row.id, title: row.title, content: row.content }; // 填充表单数据
            this.dialogVisible = true;
        },
        // 保存消息
        saveMessage() {
            if (this.isEdit) {
                // 更新消息
                this.updateMessage();
            } else {
                // 新建消息
                this.addMessage();
            }
        },
        // 处理删除消息
        handleDelete(id) {
            this.deleteMessage(id);
        }
    }
};
</script>


<style scoped>
.dialog-footer {
    text-align: right;
}
</style>
