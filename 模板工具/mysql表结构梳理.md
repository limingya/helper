# mysql表梳理
	
 - 素材
	des_material_pre_upload					预上传素材	(通过定时任务将素材同步到official_gallery)
	
	official_gallery						官方图库
	rs_des_material_des_material_kind 		素材分类关系
	des_material_kind						素材分类(二级分类)
	des_material_first_kind					素材一级分类（分类扩展分类，在二级分类上的汇总）
	
	- 素材审核
		material_verify_record				素材审核记录
	
	- 素材专题
		material_theme							素材专题表
		rs_material_theme_material_fixed_part	素材专题和素材对应表
		