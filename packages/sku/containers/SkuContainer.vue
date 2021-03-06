<template>
  <van-popup v-model="show" v-if="!isSkuEmpty" position="bottom" lockOnScroll preventScroll>
    <div class="van-sku-container">
      <div class="van-sku-layout">
        <slot name="sku-header" :skuEventBus="skuEventBus" :selectedSku="selectedSku" :selectedSkuComb="selectedSkuComb">
          <van-sku-header
            :skuEventBus="skuEventBus"
            :selectedSku="selectedSku"
            :selectedSkuComb="selectedSkuComb"
            :goods="goods"
            :sku="sku">
          </van-sku-header>
        </slot>
        <div class="van-sku-body scroller" :style="bodyStyle">
          <slot name="sku-group" :selectedSku="selectedSku" :skuEventBus="skuEventBus">
            <div v-if="hasSku" class="van-sku-group-container van-hairline--bottom">
              <div v-for="(skuTreeItem, index) in skuTree"
                class="van-sku-row-group"
                :key="index">
                <van-sku-row
                  :skuEventBus="skuEventBus"
                  :skuRow="skuTreeItem">
                  <van-sku-row-item
                    v-for="(skuValue, index) in skuTreeItem.v"
                    :key="index"
                    :skuKeyStr="skuTreeItem.k_s"
                    :skuValue="skuValue"
                    :skuEventBus="skuEventBus"
                    :selectedSku="selectedSku"
                    :skuList="sku.list">
                  </van-sku-row-item>
                </van-sku-row>
              </div>
            </div>
          </slot>
          <slot name="extra-sku-group" :skuEventBus="skuEventBus"></slot>
          <slot name="sku-stepper" :skuEventBus="skuEventBus" :selectedSku="selectedSku" :selectedSkuComb="selectedSkuComb" :selectedNum="selectedNum">
            <van-sku-stepper
              ref="skuStepper"
              :skuEventBus="skuEventBus"
              :selectedSku="selectedSku"
              :selectedSkuComb="selectedSkuComb"
              :selectedNum="selectedNum"
              :stepperTitle="stepperTitle"
              :skuStockNum="sku.stock_num"
              :quota="quota"
              :quotaUsed="quotaUsed"
              :disableStepperInput="disableStepperInput"
              :hideStock="hideStock">
            </van-sku-stepper>
          </slot>
         <slot name="sku-messages">
            <van-sku-messages
              ref="skuMessages"
              :goodsId="goodsId"
              :messagePlaceholderMap="messagePlaceholderMap"
              :messages="sku.messages">
            </van-sku-messages>
          </slot>
        </div>
        <slot name="sku-actions" :skuEventBus="skuEventBus">
          <van-sku-actions
            :skuEventBus="skuEventBus"
            :buyText="buyText"
            :showAddCartBtn="showAddCartBtn">
          </van-sku-actions>
        </slot>
      </div>
    </div>
  </van-popup>
</template>

<script>
import Vue from 'vue';
import { isServer } from '../../utils';
import Popup from '../../popup';
import Toast from '../../toast';
import SkuHeader from '../components/SkuHeader';
import SkuRow from '../components/SkuRow';
import SkuRowItem from '../components/SkuRowItem';
import SkuStepper from '../components/SkuStepper';
import SkuMessages from '../components/SkuMessages';
import SkuActions from '../components/SkuActions';
import { isAllSelected, getSkuComb, getSelectedSkuValues } from '../utils/skuHelper';
import { LIMIT_TYPE, DEFAULT_STEPPER_TITLE } from '../constants';

const { QUOTA_LIMIT } = LIMIT_TYPE;

export default {
  name: 'van-sku',

  components: {
    [Popup.name]: Popup,
    [SkuHeader.name]: SkuHeader,
    [SkuRow.name]: SkuRow,
    [SkuRowItem.name]: SkuRowItem,
    [SkuStepper.name]: SkuStepper,
    [SkuMessages.name]: SkuMessages,
    [SkuActions.name]: SkuActions
  },

  props: {
    goods: Object,
    goodsId: [Number, String],
    initialSku: {
      type: Object,
      default() {
        return {};
      }
    },
    sku: Object,
    quota: {
      type: Number,
      default: 0
    },
    quotaUsed: {
      type: Number,
      default: 0
    },
    hideStock: Boolean,
    showAddCartBtn: {
      type: Boolean,
      default: true
    },
    buyText: String,
    stepperTitle: {
      type: String,
      default: DEFAULT_STEPPER_TITLE
    },
    bodyOffsetTop: {
      type: Number,
      default: 200
    },
    resetStepperOnHide: Boolean,
    disableStepperInput: Boolean,
    messagePlaceholderMap: {
      type: Object,
      default() {
        return {};
      }
    },
    value: Boolean
  },

  data() {
    return {
      selectedSku: {},
      selectedNum: 1,
      show: this.value
    };
  },

  watch: {
    show(val) {
      this.$emit('input', val);
      if (!val) {
        const selectedSkuValues = getSelectedSkuValues(this.sku.tree, this.selectedSku);

        this.$emit('sku-close', {
          selectedSkuValues,
          selectedNum: this.selectedNum,
          selectedSkuComb: this.selectedSkuComb
        });

        if (this.resetStepperOnHide) {
          this.$refs.skuStepper && this.$refs.skuStepper.setCurrentNum(1);
        }
      }
    },
    value(val) {
      this.show = val;
    },
    skuTree(val) {
      this.resetSelectedSku(val);
    }
  },

  computed: {
    bodyStyle() {
      if (isServer) {
        return;
      }

      const windowHeight = window.innerHeight;
      // header高度82px, sku actions高度50px，如果改动了样式自己传下bodyOffsetTop调整下
      const maxHeight = windowHeight - this.bodyOffsetTop;

      return {
        maxHeight: maxHeight + 'px'
      };
    },
    isSkuCombSelected() {
      return isAllSelected(this.sku.tree, this.selectedSku);
    },
    // sku数据不存在时不渲染模板
    isSkuEmpty() {
      for (var key in this.sku) {
        if (Object.prototype.hasOwnProperty.call(this.sku, key)) return false;
      }
      return true;
    },
    hasSku() {
      return !this.sku.none_sku;
    },
    selectedSkuComb() {
      if (!this.hasSku) {
        return {
          id: this.sku.collection_id,
          price: Math.round(this.sku.price * 100),
          'stock_num': this.sku.stock_num
        };
      } else if (this.isSkuCombSelected) {
        return getSkuComb(this.sku.list, this.selectedSku);
      }
      return null;
    },
    skuTree() {
      return this.sku.tree || [];
    }
  },

  created() {
    var skuEventBus = new Vue();
    this.skuEventBus = skuEventBus;

    skuEventBus.$on('sku:close', this.handleCloseClicked);
    skuEventBus.$on('sku:select', this.handleSkuSelected);
    skuEventBus.$on('sku:numChange', this.handleNumChange);
    skuEventBus.$on('sku:overLimit', this.handleOverLimit);
    skuEventBus.$on('sku:addCart', this.handleAddCartClicked);
    skuEventBus.$on('sku:buy', this.handleBuyClicked);

    this.resetSelectedSku(this.skuTree);
    // 组件初始化后的钩子，抛出skuEventBus
    this.$emit('after-sku-create', skuEventBus);
  },

  methods: {
    resetSelectedSku(skuTree) {
      this.selectedSku = {};
      skuTree.forEach((item) => {
        // 只有一个sku规格值时默认选中
        if (item.v.length === 1) {
          this.selectedSku[item.k_s] = item.v[0].id;
        } else {
          this.selectedSku[item.k_s] = this.initialSku[item.k_s] || '';
        }
      });
    },
    getSkuMessages() {
      return this.$refs.skuMessages
        ? this.$refs.skuMessages.getMessages()
        : {};
    },
    getSkuCartMessages() {
      return this.$refs.skuMessages
        ? this.$refs.skuMessages.getCartMessages()
        : {};
    },
    validateSkuMessages() {
      return this.$refs.skuMessages
        ? this.$refs.skuMessages.validateMessages()
        : '';
    },
    validateSku() {
      if (this.selectedNum === 0) {
        return '商品已经无法购买啦';
      }

      if (this.isSkuCombSelected) {
        const error = this.validateSkuMessages();
        // sku留言没有错误则校验通过
        return error;
      } else {
        return '请选择完整的规格';
      }
    },
    handleCloseClicked() {
      this.show = false;
    },
    handleSkuSelected(skuValue) {
      // 点击已选中的sku时则取消选中
      this.selectedSku = this.selectedSku[skuValue.skuKeyStr] === skuValue.id
        ? { ...this.selectedSku, [skuValue.skuKeyStr]: '' }
        : { ...this.selectedSku, [skuValue.skuKeyStr]: skuValue.id };

      this.$emit('sku-selected', {
        skuValue,
        selectedSku: this.selectedSku,
        selectedSkuComb: this.selectedSkuComb
      });
    },
    handleNumChange(num) {
      this.selectedNum = num;
    },
    handleOverLimit({ action, limitType, quota, quotaUsed }) {
      if (action === 'minus') {
        Toast('至少选择一件');
      } else if (action === 'plus') {
        if (limitType === QUOTA_LIMIT) {
          let msg = `限购${quota}件`;
          if (quotaUsed > 0) msg += `，您已购买${quotaUsed}件`;
          Toast(msg);
        } else {
          Toast('库存不足');
        }
      }
    },
    handleAddCartClicked() {
      this.handleBuyOrAddCart('add-cart');
    },
    handleBuyClicked() {
      this.handleBuyOrAddCart('buy-clicked');
    },
    handleBuyOrAddCart(type) {
      const error = this.validateSku();
      if (error) {
        Toast(error);
        return;
      }
      this.$emit(type, {
        goodsId: this.goodsId,
        selectedNum: this.selectedNum,
        selectedSkuComb: this.selectedSkuComb,
        messages: this.getSkuMessages(),
        cartMessages: this.getSkuCartMessages()
      });
    }
  }
};
</script>
