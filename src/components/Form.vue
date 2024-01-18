
<template>
  <div class="helpTips">
    <el-link target="_blank" href="https://ywlbw2ffwk.feishu.cn/docx/JD0GdsUmhoE0agxTI3Sc3iWPn6z?from=from_copylink">
      帮助文档
    </el-link>
  </div>
  <el-form ref="form" class="form" :model="formData" label-position="left">
    <el-form-item label="选择操作："  size="large" :rules="{ required: true, message: '请选择操作类型' }">
      <el-select v-model="formData.choosedOperation" placeholder="请选择SQL操作类型" style="width: 100%">
        <el-option
          v-for="meta in operationTypeList"
          :key="meta.key"
          :label="meta.value"
          :value="meta.key"
        />
      </el-select>
    </el-form-item>
    <el-form-item label="操作表名："  size="large" :rules="{ required: true, message: '请输入操作表名' }">
      <el-input v-model="formData.tableName" placeholder="请输入操作数据库表的名称"></el-input>
    </el-form-item>
    <el-form-item label="选择主键：" v-if="formData.choosedOperation === 'update'" size="large" :rules="{ required: true, message: '请选择主键字段' }">
        <el-select v-model="formData.choosedPrimaryField" @change="" placeholder="请选择主键字段" style="width: 100%">
          <el-option
            v-for="meta in fieldMetaList"
            :key="meta.name"
            :label="meta.name"
            :value="meta.name"
          />
        </el-select>
    </el-form-item>
    <el-form-item label="处理字段：" size="large" :rules="{ required: true, message: '请选择处理字段' }">
        <el-select v-model="formData.choosedFieldList" style="width: 100%" multiple placeholder="请选择待生成SQL的字段">
          <el-option
            v-for="item in fieldMetaList"
            :key="item.name"
            :label="item.name"
            :value="item.name">
          </el-option>
        </el-select>
    </el-form-item>
    <el-button type="primary" style="width: 100%" size="large" @click="addColumn">生成SQL</el-button>
  </el-form>
</template>

<script>
  import { bitable,FieldType } from '@lark-base-open/js-sdk';
  import { ref, onMounted } from 'vue';  

  export default {
    setup() {
      //表单信息
      const formData = ref({});
      //数据表字段信息
      const fieldMetaList = ref([]);
      //初始化操作类型
      const operationTypeList = ref([{"key":"insert", "value":"插入"},{"key":"update", "value":"更新"},]);
      //新增字段名称
      const addColumnName = "sql脚本_自动生成";
      //新增字段操作
      const addColumn = async () => {
        //验证表单信息
        if(!formData.value.choosedOperation){
          ElMessage({
            message: '请选择SQL操作类型！',
            type: 'warning',
          })
          return;
        }
        if(!formData.value.tableName){
          ElMessage({
            message: '请输入操作数据库表的名称！',
            type: 'warning',
          })
          return;
        }
        if(formData.value.choosedFieldList.length == 0){
          ElMessage({
            message: '请选择生成SQL字段！',
            type: 'warning',
          })
          return;
        }

        //如果是update操作，必须要有主键
        if(formData.value.choosedOperation == "update" && !formData.value.choosedPrimaryField){
          ElMessage({
            message: '请选择主键字段！',
            type: 'warning',
          })
          return;
        }
        
        //获取当前数据表信息
        const tableId = formData.value.activeTableId;
        if (tableId) {
          //获取当前数据库表对象
          const table = await bitable.base.getTableById(tableId);
          const tableFieldMetaList = await table.getFieldMetaList();
          //判断当前数据表下是否已存在 addColumnName 字段
          const fieldInfo = tableFieldMetaList.find(field => field.name === addColumnName);
          var fieldId;
          if(fieldInfo){
            fieldId = fieldInfo.id;
          }else{
            //新增sql自动生成列
            fieldId = await table.addField({ type: FieldType.Text,name:addColumnName});
          }
          const field = await table.getField(fieldId);
          const recordList = await table.getRecordList();
          for (const record of recordList) {
            const updateCell = await record.getCellByField(fieldId);
            var sqlStr ="";
            if(formData.value.choosedOperation == "update"){ 
              sqlStr ="update "+formData.value.tableName+" set ";
              for(const fieldName of formData.value.choosedFieldList){
                const field = await table.getFieldByName(fieldName);
                //获取单元格的值
                const cellStrValue =  await table.getCellString(field.id,record.id);
                if(cellStrValue !== null){
                  sqlStr = sqlStr + fieldName + "='"+cellStrValue+"',";
                }else{
                  sqlStr = sqlStr + fieldName + "= null,";
                }
                
              }
              //移除最后一个逗号
              sqlStr = sqlStr.substring(0, sqlStr.length - 1);
              //拼接最后的where信息
              const field = await table.getFieldByName(formData.value.choosedPrimaryField);
              //获取单元格的值
              const cellStrValue =  await table.getCellString(field.id,record.id);
              if(cellStrValue !==null){
                sqlStr = sqlStr +" where "+ formData.value.choosedPrimaryField+"='"+cellStrValue+"';";
                updateCell.setValue(sqlStr);
              }
            }else{
              sqlStr ="insert into "+formData.value.tableName+"(";
              for(const fieldName of formData.value.choosedFieldList){
                sqlStr = sqlStr+fieldName+",";
              }
              //移除最后一个逗号
              sqlStr = sqlStr.substring(0, sqlStr.length - 1);
              sqlStr = sqlStr + ") values(";
              for(const fieldName of formData.value.choosedFieldList){
                const field = await table.getFieldByName(fieldName);
                console.log(field)
                //获取单元格的值
                const cellStrValue =  await table.getCellString(field.id,record.id);
                if(cellStrValue !== null){
                  sqlStr = sqlStr  + "'"+cellStrValue+"',";
                }else{
                  sqlStr = sqlStr  + "null,";
                }
              }
              //移除最后一个逗号
              sqlStr = sqlStr.substring(0, sqlStr.length - 1);
              sqlStr = sqlStr + ");";
              updateCell.setValue(sqlStr);
            }
          }
          //提示数据处理完毕
          ElMessage({
            message: 'SQL脚本生成完毕！',
            type: 'success',
          })
        }
      };
      onMounted(async () => {
        //获取到当前选中的数据表对象
        const table = await bitable.base.getActiveTable();
        formData.value.activeTableId = table.id;
        //获取数据表下面所有的字段信息 
        const fieldList = await table.getFieldMetaList();
        fieldMetaList.value = fieldList;
      });

      return {
        formData,
        fieldMetaList,
        addColumn,
        operationTypeList,
      };
    },
  };
</script>


<style scoped>
  .form :deep(.el-form-item__label) {
    font-size: 16px;
    color: var(--el-text-color-primary);
    margin-bottom: 0;
  }
  .form :deep(.el-form-item__content) {
    font-size: 16px;
  }

  .tips{
    font-size:12px;
    color: #999;
    margin-bottom:10px;
  }
  .helpTips{
    text-align:right;
    margin-bottom:20px;
    padding-right:10px;
  }
</style>
