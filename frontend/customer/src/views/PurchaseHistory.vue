<template>
	<div class="purchase-history">
		<h2 class="page-title">Lịch sử mua hàng</h2>

		<div v-if="loading" class="loading-state">
			<Loading />
		</div>

		<div v-else-if="error" class="error-state">
			<i class="fas fa-exclamation-circle"></i> Đã xảy ra lỗi khi tải lịch sử mua hàng. Vui lòng thử lại sau.
		</div>

		<div v-else-if="historyOrders.length === 0" class="empty-state">
			<i class="fas fa-box-open"></i> Bạn chưa có đơn hàng nào trong lịch sử.
		</div>

		<div v-else class="history-table-container">
			<table class="purchase-table">
				<thead>
					<tr>
						<!-- <th>Mã ĐH</th> -->
						<th>Ngày mua</th>
						<th>Sản phẩm</th>
						<th>Tổng tiền</th>
						<th>Trạng thái</th>
						<th>Hành động</th>
					</tr>
				</thead>
				<tbody>
					<tr v-for="order in historyOrders" :key="order.id">
						<!-- <td data-label="Mã ĐH" class="order-id-cell">#{{ getOrderID(order.id) }}</td> -->
						<td data-label="Ngày mua">{{ format.formatDate(order.orderDate) }}</td>
						<td data-label="Sản phẩm">
							<div class="product-list-cell">
								<div v-for="item in order.orderItems.slice(0, 1)" :key="item.id" class="product-item">
									<img :src=productService.getUrlImage(item.colorImage) :alt="item.name" class="product-thumb-small" />
									<span class="product-name">{{ item.product.name }} - {{ item.colorName }}(x{{ item.quantity }})</span>
								</div>
								<p v-if="order.orderItems.length > 2" class="more-items-in-table">
									và {{ order.orderItems.length - 1 }} sản phẩm khác...
								</p>
							</div>
						</td>
						<td data-label="Tổng tiền" class="total-amount-cell">{{ format.formatPrice(order.totalAmount) }}₫</td>
						<td data-label="Trạng thái">
							<span :class="['status-badge','btn', getStatusClass(order.status)]">{{ order.status }}</span>
						</td>
						<td data-label="Hành động">
							<div class="action-buttons">
								<button @click="showPopup(order)" class="btn btn-primary">Chi tiết</button>
								<button @click="reviewOrder(order.id)" class="btn btn-review">Đánh giá</button>
							</div>
						</td>
					</tr>
				</tbody>
			</table>
		</div>
		<div v-if="popupVisible" class="popup-overlay" @click.self="closePopup">
			<div class="popup-content">
				<h3>Chi tiết đơn hàng #{{ getOrderID(selectedOrder.id) }}</h3>
				<p><strong>Ngày đặt:</strong> {{ format.formatDate(selectedOrder.orderDate) }}</p>
				<p><strong>Trạng thái:</strong> {{ selectedOrder.status }}</p>
				<p><strong>Tổng tiền:</strong> {{ format.formatPrice(selectedOrder.totalAmount) }} ₫</p>

				<h4>Sản phẩm:</h4>
				<ul class="popup-product-list">
				<li v-for="item in selectedOrder.orderItems" :key="item.id" class="popup-product-item">
					<img :src=productService.getUrlImage(item.colorImage) :alt="item.product.name" class="popup-product-thumbnail" />
					<div>
					<p class="popup-product-name">{{ item.product.name }}</p>
					<p>Màu sắc: {{ item.colorName }}</p>
					<p>Số lượng: {{ item.quantity }}</p>
					<p>Giá: {{ format.formatPrice(item.price) }} ₫</p>
					</div>
				</li>
				</ul>
				<div class="container-btn">
				<button @click="closePopup" class="btn btn-close">Đóng</button>
				</div>
			</div>
		</div>
		<Pagi
          :totalProducts="totalOrders"
          :currentPage="currentPage"
          :pageSize="pageSize"
          @pageChanged="getOrdersInPage"
        />
	</div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import Loading from '@/components/common/Loading.vue';
import productService from '@/services/productService';
import format from '@/utils/format';
import Pagi from '@/components/common/Pagination.vue';

const router = useRouter();

const historyOrders = ref([]);
const allHistoryOrders = ref([]);
const loading = ref(true);
const error = ref(false);
const popupVisible = ref(false);
const selectedOrder = ref(null);
const pageSize = ref(10); 
const currentPage = ref(1);
const totalOrders = ref(0); 

const closePopup = () => {
	popupVisible.value = false;
	selectedOrder.value = null;
};

const getOrdersInPage = (page) => {
	currentPage.value = page;
	historyOrders.value = allHistoryOrders.value.slice((currentPage.value - 1) * pageSize.value, currentPage.value * pageSize.value);
	totalOrders.value = allHistoryOrders.value.length;
	window.scrollTo({ top: 0, behavior: 'smooth' });
};
const showPopup = (order) => {
  selectedOrder.value = order;
  popupVisible.value = true;
};
const fetchHistoryOrders = async () => {
	loading.value = true;
	error.value = false;
	const tmp = await productService.getAllOrders()
		.catch(err => {
			console.error("Lỗi khi lấy dữ liệu lịch sử mua hàng:", err);
			error.value = true;
		})
		.finally(() => {
			loading.value = false;
		});
	console.log("Lấy dữ liệu lịch sử mua hàng:", tmp);
	allHistoryOrders.value = tmp.filter(order => ['Hoàn thành', 'Đã trả hàng', "Đã huỷ"].includes(order.status));
	console.log("Lấy dữ liệu lịch sử mua hàng:", historyOrders.value);
	historyOrders.value = allHistoryOrders.value.slice(0, pageSize.value);
	totalOrders.value = allHistoryOrders.value.length;
	
};

const getOrderID = (order) => {
  return order.toString().substring(0,8);
};

// Hàm trả về class cho trạng thái để custom CSS
const getStatusClass = (status) => {
	switch (status) {
		case 'Hoàn thành': return 'status-delivered';
		case "Đã huỷ": return 'status-cancelled';
		case 'Đã trả hàng': return 'status-refunded';
		default: return '';
	}
};



// Xử lý đánh giá đơn hàng/sản phẩm
const reviewOrder = (orderId) => {
	alert(`Chức năng đánh giá cho đơn hàng ${orderId} sẽ sớm ra mắt!`);
	console.log("Đánh giá đơn hàng:", orderId);
	// Có thể điều hướng đến trang đánh giá
	// router.push({ name: 'ReviewOrder', params: { id: orderId } });
};



onMounted(async () => {
	await fetchHistoryOrders();
});
</script>

<style scoped>
.purchase-history {
  margin: 0 auto;
  padding: 2rem;
  color: #333;
    min-height: calc(100vh - 100px);
  background-color: white;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 1.5rem;
}

.page-title {
  font-weight: 600;
  text-shadow: 1px 1px 2px rgba(214, 51, 132, 0.1);
  position: relative;
  margin-bottom: 1.5rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #eee;
}

.page-title::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  width: 7rem;
  height: 2px;
  background-color: #060606;
  border-radius: 10px;
}

/* Loading & Empty States */
.loading-state,
.error-state,
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 3rem;
  text-align: center;
  background-color: #fff;
  border-radius: 10px;
  box-shadow: 0 2px 15px rgba(0, 0, 0, 0.05);
  min-height: 300px;
}

.error-state i,
.empty-state i {
  font-size: 3rem;
  margin-bottom: 1rem;
  color: var(--primary-color);
}

.error-state {
  color: #d32f2f;
}

.empty-state {
  color: var(--primary-color);
}

/* Table Container */
.history-table-container {
  overflow-x: auto;
  background: #fff;
  border-radius: 10px;
  box-shadow: 0 2px 20px rgba(0, 0, 0, 0.08);
  /* padding: 1rem; */
  margin-top: 1.5rem;
}

.purchase-table {
  width: 100%;
  border-collapse: collapse;
  min-width: 800px;
}

.purchase-table th {
  background-color: var(--primary-color);
  color: white;
  padding: 1rem;
  text-align: left;
  font-weight: 500;
}

.purchase-table td {
  padding: 1rem;
  border-bottom: 1px solid #f0f0f0;
  vertical-align: middle;
}

.purchase-table tr:last-child td {
  border-bottom: none;
}

.purchase-table tr:hover {
  background-color: #fff9f9;
}

/* Table Cells */
.order-id-cell {
  font-weight: 600;
  color: var(--primary-color);
}

.product-list-cell {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.product-item {
  display: flex;
  align-items: center;
  gap: 0.8rem;
}

.product-thumb-small {
  width: 50px;
  height: 50px;
  object-fit: cover;
  border-radius: 5px;
  border: 1px solid #eee;
}

.product-name {
  font-size: 0.9rem;
  line-height: 1.4;
}

.more-items-in-table {
  font-size: 0.8rem;
  color: #888;
  margin-top: 0.3rem;
}

.total-amount-cell {
  font-weight: 600;
  color: var(--primary-color);
}

/* Status Badges */
.status-badge {
  text-align: center;
  padding: 0.4rem 0.8rem !important;
  border-radius: 20px !important;
  font-size: 0.8rem;
  font-weight: 500;
  text-transform: capitalize;
}

.status-delivered {
  background-color: #e8f5e9;
  color: #2e7d32;
}

.status-cancelled {
  background-color: #ffebee;
  color: #c62828;
}

.status-refunded {
  background-color: #e3f2fd;
  color: #1565c0;
}

/* Action Buttons */
.action-buttons {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 5px;
  font-size: 0.85rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.3rem;
}

.btn-primary {
  background-color: var(--primary-color);
  color: white;
}

.btn-primary:hover {
  background-color: #e1618c;
  transform: translateY(-2px);
}

.btn-review {
  background-color: #f8bbd0;
  color: #ad1457;
}

.btn-review:hover {
  background-color: #f48fb1;
  transform: translateY(-2px);
}

/* Popup Styles */
.popup-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  backdrop-filter: blur(3px);
}

.popup-content {
  background-color: white;
  border-radius: 10px;
  padding: 2rem;
  max-width: 600px;
  width: 90%;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 5px 30px rgba(0, 0, 0, 0.2);
  animation: popupFadeIn 0.3s ease-out;
}

@keyframes popupFadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.popup-content h3 {
  color: var(--primary-color);
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
  border-bottom: 1px solid #f0f0f0;
  padding-bottom: 0.5rem;
}

.popup-content p {
  margin-bottom: 0.8rem;
  line-height: 1.6;
}

.popup-content h4 {
  margin: 1.5rem 0 1rem;
  color: var(--primary-color);
  font-size: 1.2rem;
}

.popup-product-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.popup-product-item {
  display: flex;
  gap: 1rem;
  padding: 1rem 0;
  border-bottom: 1px solid #f5f5f5;
  align-items: center;
}

.popup-product-item:last-child {
  border-bottom: none;
}

.popup-product-thumbnail {
  width: 70px;
  height: 70px;
  object-fit: cover;
  border-radius: 5px;
  border: 1px solid #eee;
}

.popup-product-name {
  font-weight: 600;
  margin-bottom: 0.3rem;
  color: #333;
}

.container-btn {
  display: flex;
  justify-content: flex-end;
  margin-top: 1.5rem;
}

.btn-close {
  background-color: #f5f5f5;
  color: #666;
}

.btn-close:hover {
  background-color: #eee;
}

/* Responsive Table */
@media (max-width: 768px) {
  .purchase-table {
    display: block;
  }
  
  .purchase-table thead {
    display: none;
  }
  
  .purchase-table tr {
    display: block;
    margin-bottom: 1.5rem;
    border: 1px solid #f0f0f0;
    border-radius: 8px;
    padding: 0.5rem;
  }
  
  .purchase-table td {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0.5rem 1rem;
    border-bottom: 1px solid #f5f5f5;
  }
  
  .purchase-table td:last-child {
    border-bottom: none;
  }
  
  .purchase-table td::before {
    content: attr(data-label);
    font-weight: 600;
    color: var(--primary-color);
    margin-right: 1rem;
  }
  
  .action-buttons {
    justify-content: flex-end;
  }
  
  .product-list-cell {
    flex-direction: column;
    align-items: flex-start;
  }
}
.container-btn{
  text-align: end;
}

.btn-close {
  text-align: end;
  background-color: var(--primary-color);
  color: white;
  padding: 0.6rem 1.6rem;
  border-radius: 30px;
  font-weight: 600;
  font-size: 1.1rem;
  cursor: pointer;
  user-select: none;
  transition: background-color 0.3s ease;
}

.btn-close:hover {
  background-color: #eb2ddb;
}
</style>